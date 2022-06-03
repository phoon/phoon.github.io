# Chdir()与Chroot()






最近在学习容器时，遇到了以下几个系统调用，遂作个记录

- chdir：更改工作目录
- chroot：更改root目录

### chdir
```c
#include <unistd.h>
int chdir(const char *path);
```
#### 示例
chdir将当前目录改为当前目录下的test目录,而fork出的子程序也继承了这个目录
```c
/* chdir.c */

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
int main(void)
{
  pid_t pid;
  
  if(chdir("./test") != 0) {
    printf("chdir error\n");
  }
  pid = fork();
  if(pid == 0) {
    printf("Child dir: %s\n", get_current_dir_name());
  }else {
    printf("Parent dir: %s\n", get_current_dir_name());
  }
  return 0;
}
```
### chroot
每一个进程都有一个根目录，一般是对应文件系统的真实根目录，且每个进程都会继承其父进程的根目录。可以通过`chroot()`来改变一个进程的根目录。此后，所有绝对路径的解释都将从该目录开始（即假装自己是那个`/`）。`chroot()`并不改变当前工作目录。
```c
#include <unistd.h>
int chroot(const char *path);
```
>`chroot()`必须与`chdir()`一同使用，否则程序将有可能通过使用相对路径来进行“越狱”。为避免“越狱”情况的发生，同时还应该将程序中之前打开的“狱外”文件都关闭，因为使用`fchdir()`可以通过之前打开的文件描述符而达到越狱的目的。

#### 示例
```c
/* chroot.c */

#include <stdio.h>
#include <unistd.h>

int main(void)
{
    FILE *f;

    /*
     *  Or:
     *  chdir("./jail");
     *  chroot("./");
     */

    if(chroot("./jail") != 0) {
        perror("chroot() error");
    }

    if(chdir("/") != 0) {
        perror("chdir() error");
    }
    /* do something after chrooting */
    f = fopen("/in_jail.txt", "r");

    if(f == NULL) {
        perror("/in_jail.txt error");
    }else {
        char buf[100];
        while(fgets(buf, sizeof(buf), f)) {
            printf("%s\n", buf);
        }
    }

    return 0;
}

```
我的目录结构：
```bash
.
|-- chroot.c
|-- chroot.o
|-- out_of_jail.txt
`-- jail
    `-- in_jail.txt
```
再看一个越狱的例子：
```c
/* chroot_breaking_out_jial.c */

#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
int main(void)
{
    int f1;
    FILE *f2;
    f1 = open("/", O_RDONLY);

    if(chroot("./test") != 0) {
        perror("chroot() error");
    }

    if(chdir("/") != 0) {
        perror("chdir() error");
    }
    /* close(f1); */
    /* Breaking out the jail */
    fchdir(f1);
    chroot(".");
    printf("%s\n", get_current_dir_name());
    f2 = fopen("/root/WORK/out_of_jail.txt", "r");
    if(f2 == NULL) {
        perror("out_of_jail_txt error");
    }else {
        char buf[100];
        while(fgets(buf, sizeof(buf), f2)) {
            printf("%s\n", buf);
        }
    }
    return 0;
}
```
