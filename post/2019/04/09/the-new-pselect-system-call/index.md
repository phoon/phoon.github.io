# [译]pselect()系统调用


`译自 [The new pselect() system call] https://lwn.net/Articles/176911/`

像网络服务器等需要使用`select()`，`poll()`，或者`epoll_wait()`(Linux独有)等系统调用来对多个文件描述符进行监视的应用程序有时会面临这样一个问题：该如何等待其中的一个文件描述符变为`ready`状态，或者是接收一个signal（比如`SIGINT`）。因为事实表明，这些系统调用不能很好地与信号进行交互。

一个看似显而易见的解决方案是写一个空的信号处理器，以便传递信号从而中断`select()`的调用。

```c
static void handler(int sig) {}
    
int main(int argc, char *argv[])
{
	fd_set readfds;
	struct sigaction sa;
	int nfds, ready;

	sa.sa_handler = handler;     /* Establish signal handler */
	sigemptyset(&sa.sa_mask);
	sa.sa_flags = 0;
	sigaction(SIGINT, &sa, NULL);
	/* ... */    
	ready = select(nfds, &readfds, NULL, NULL, NULL);
	/* ... */

```

待到`select()`返回，我们就可以通过查看其返回值以及`errno`来确定发生了什么。如果`errno`是`EINTR`，我们就知道`select()`的调用被信号给中断了，并做出了相应的行动。但这种解决方案存在竞态条件：如果信号`SIGINT`是在调用了`sigaction()`函数之后，但在调用`select()`之前传递过去的，那么它将无法中断`select()`调用，因为这时候信号已经丢失了。

我们也可以尝试其他的方案，比如在信号处理器里设置一个全局标志，然后在主程序里监控这个标志，并使用`sigprocmask()`来阻塞那个信号直到调用了`select()`。然而，这些技术都不能完全消除竞态条件：因为在`select()`调用开始之前，总是有一段时间间隔（无论其有多么短）可以处理信号。

这个问题的传统解决方案是所谓的`自管(self-pipe)`技巧，通常认为是由[D J Bernstein](http://cr.yp.to/docs/selfpipe.html)首创的。使用这个技术，程序建立一个信号处理器，该信号处理器将一个字节写入一个特别建立的管道之中，该管道的读取端也将由`select()`进行监控。`self-pipe`技术很巧妙的解决了安全的等待文件描述符变为`ready`状态或者需要传递信号的问题。然而，这需要通过大量的代码来解决一个实质上很简单的需求。（例如，一个健壮的解决方案需要同时标记管道的读写端）

出于此因，`POSIX.1g`委员会设计了一个`select()`的增强版本，也就是`pselect()`。`select()`与`pselect()`之间最主要的区别就是后者有一个`sigset_t`类型的信号掩码（sigmask）作为额外的参数：

```c
int pselect(int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
            const struct timespec *timeout,
            const sigset_t *sigmask);
```

`sigmask`参数指定了一个应该在`pselect()`调用期间阻塞的信号集合，它会在调用期间覆盖当前的信号掩码。当我们做以下调用时：

```c
 ready = pselect(nfds, &readfds, &writefds, &exceptfds, timeout, &sigmask);
```

内核会执行一系列步骤，这相当于原子地执行以下系统调用：

```c
sigset_t sigsaved;

sigprocmask(SIG_SETMASK, &sigmask, &sigsaved);
ready = select(nfds, &readfds, &writefds, &exceptfds, timeout);
sigprocmask(SIG_SETMASK, &sigsaved, NULL);
```

`glibc`虽也提供了`pselect()`的库实现，而且实际上也是使用了上面的系统调用序列。但问题是，这种实现依然很容易受到`pselect()`用来避免竞态条件的设计的影响，因为单独的系统调用并不是原子的。

使用`pselect()`，我们可以安全地等待信号传递或者文件描述符进入ready状态，只需将我们前面的示例代码修改为：

```c
sigset_t emptyset, blockset;

sigemptyset(&blockset);         /* Block SIGINT */
sigaddset(&blockset, SIGINT);
sigprocmask(SIG_BLOCK, &blockset, NULL);

sa.sa_handler = handler;        /* Establish signal handler */
sa.sa_flags = 0;
sigemptyset(&sa.sa_mask);
sigaction(SIGINT, &sa, NULL);
    
/* Initialize nfds and readfds, and perhaps do other work here */
/* Unblock signal, then wait for signal or ready file descriptor */

sigemptyset(&emptyset);
ready = pselect(nfds, &readfds, NULL, NULL, NULL, &emptyset);
... 
```

这段代码可以工作，因为`SIGINT`信号只有在控制传递给内核之后才会被解除阻塞。因此，在`pselect()`执行之前，无法传递信号。如果信号是在`pselect()`被阻塞时生成的，那么与`select()`一样，系统调用将被中断，且信号将在系统调用返回之前传递完成。

虽然`pselect()`是在几年前构思的，也已经在1998由[W. Richard Stevens](http://www.kohala.com/start/)在其著作*Unix Network Programming, vol. 1, 2nd ed.*中公布，但实际的实现却很晚才出现。随着2.6.16版本的释出，以及更新的`glibc`2.4，Linux才可以使用`pselect()`。

Linux 2.6.16同时也包含了一个新的（但不是标准的）`ppoll()`系统调用，它也是在传统的`poll()`接口中增加了一个信号掩码参数：

```c
int ppoll(struct pollfd *fds, nfds_t nfds, const struct timespec *timeout, 
             const sigset_t *sigmask);
```

`ppoll()`增添了与`pselect()`相对于`select()`相同的功能。为了不受冷落，`epoll`维护者在`pipeline`中添加了补丁，以新的`epoll_pwait()`系统调用的形式添加类似的功能。

`pselect()`和`ppoll()`与传统方法相比还有一些其他的细微差别。例如超时的类型为:

```c
struct timespec {
	long tv_sec;        /* Seconds */
	long tv_nsec;       /* Nanoseconds */
};
```

这允许用比以前的系统调用更高的精度来指定超时间隔。

`pselect()`和`select()`的`glibc` wrappers还隐藏了一些底层系统调用细节:

首先，系统调用实际上期望信号掩码由两个参数来进行描述，其中一个参数是一个指向`sigset_t`结构的指针，而另一个参数是一个以字节表示该结构大小的整数。这使得将来可能会有更大的`sigset_t`类型。

底层系统调用还修改了他们的超时参数，以便在函数提前返回时（文件描述符进入`ready`状态，或者传递了信号），调用方能够知道剩余的超时时间。但是，各个wrapper函数通过创建超时参数的本地副本并将该副本传递给底层系统调用来隐藏其详情。（Linux的`select()`也修改了其超时参数，并且对应用程序是可见的。不过许多其他的`select()`实现并没有修改此参数。`POSIX.1`承认他们之中的任何一种实现。）

有关`pselect()`和`ppoll()`的详细信息可以在man pages中查询`select(2)`和`poll(2)`。



原文：

[The new pselect() system call - [LWN.net]](https://lwn.net/Articles/176911/)
