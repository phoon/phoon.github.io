# InnoDB Doublewrite机制


## 楔子

最近再次翻看《InnoDB存储引擎》，看到`Doublewrite`时，直觉告诉我这玩意儿应该很简单（其实确实是），但总感觉书上的文字表达不够清晰，再加上这背后涉及到的底层知识其实也挺有趣的，遂各处寻解并作此文以记录。内容出于个人理解，有错误的地方还请指正。

## InnoDB逻辑存储结构

在正式开始介绍`Doublewrite`技术前，先来粗略的看一下`InnoDB`存储引擎的逻辑存储结构：

{{<image src="https://dd-static.jd.com/ddimg/jfs/t1/207714/7/3802/38482/615d3929E75f1e719/79d3ccb5d940b1a1.webp" width=100% caption="InnoDB逻辑存储结构">}}

`InnoDB`中所有的数据都逻辑地存放在表空间（Tablespace）中，且默认会有一个共享的表空间`ibdata1`（若启用了`innodb_file_per_table`参数，则每张表内的数据可单独放于一个表空间内）；表空间用来存放页（Page），也即`InnoDB`引擎进行磁盘管理的最小单位，页内按行存储了我们的实际数据，页大小由`innodb_page_size`参数设置，默认为16KB；一定数量的连续的页面会组成一个区（Extent），而区的大小则有以下规则限定：

| Page Size | Consecutive Pages Count | Extent Size |
| :-------: | :---------------------: | :---------: |
|    4KB    |           256           |     1MB     |
|    8KB    |           128           |     1MB     |
|   16KB    |           64            |     1MB     |
|   32KB    |           64            |     2MB     |
|   64KB    |           64            |     4MB     |

进而由多个区所构成的表空间文件叫做段（Segment），如数据段（图中`Leaf-node segment`）、索引段（图中`Non-Leaf-node segment`）、回滚段（图中`Rollback segment`，但注意：与其它段不同，回滚段里包含了其它的表空间段）等。

对于`InnoDB`存储引擎的逻辑存储结构先介绍到这里，接下来我们来看`InnoDB`存储引擎里非常重要的`Doublewrite`特性。

## Doublewrite机制

### 页断裂问题

页断裂，亦称部分写失效（partial page write），描述的是这样一个问题：恰逢系统宕机时，`InnoDB`正在更新/写入一个页的数据，可能会导致数据只完成部分写入，而剩余的部分的数据则是无效数据或者陈旧数据。

发生这个问题的原因在于，对于`InnoDB`页的更新/写入不具备原子性。不具备原子性是因为：太大了（`InnoDB`页大小最小设置都得4KB，当然太小也不可取，这里仅为说明`页断裂`戏谑几句）。不像重做日志（redo log），`redo-log`的块大小为512字节，与磁盘扇区大小一致——对于磁盘来说，使用扇区作为最小传输单位，传统上扇区长度即为512字节。所以`redo-log`的写入可以保证原子性：因为从硬件设计上来说，现代硬盘自身的电量会足够完成断电后一个扇区的写入操作，以此保证原子性。

而发生页断裂后，要进行修复的前提是，我们得有一份该页的副本。`InnoDB`默认会在加载页时对其进行完整性校验（Page Header和Page Trailer中记录了LSN数据），校验未通过则表明该页可能有页断裂问题，这时需要对该页进行恢复。有的人疑惑为什么这里不能通过redo log来对断裂页进行修复，因为redo log是根据页中的LSN来判断页是否需要进行恢复，换句话说，redo log根本解决不了页断裂的问题，毕竟该页面已经损坏了，无法提供信息给redo log参考。所以我们得重新写入该页面，而在哪里重新再获取到这个页面呢？

### 解决之道

为了解决上面提到的问题，`InnoDB`引入了`Doublewrite`机制。其作用就是保存一份最近更新过的页的副本，方便发生页断裂时进行页恢复，从而保证在遇到系统宕机时写页面的原子性和持久性。由局部性（程序通常倾向于引用临近于其他最近引用过的数据项的数据项，抑或最近引用过的数据项本身）可知，这样的机制应该会有不错的效果表现。

{{<image src="https://dd-static.jd.com/ddimg/jfs/t1/202757/10/10048/20616/615dada9Ee06b7ace/37559b99a206d04e.webp" width=100% caption="doublewrite架构">}}

`Doublewrite`由两部分构成：内存中的`doublewrite buffer`以及共享表空间里的2个区（128个连续的页），它们的总大小都是2MB。

这里使用两个Extent分两次写入还有一个用处：双重缓冲。当在对一个区进行写入时，对应的另一个区的`Doublewrite buffer`还可以继续缓冲待flush的脏页。

在对内存缓冲池中的脏页进行flush时的步骤如下（此前对页的更改已经记录至redo log）：

- 不直接写磁盘，先拷贝至`doublewrite buffer`；
- `doublewrite buffer`分两次，每次1MB顺序地写入到共享表空间，随后立即调用fsync确保落盘；
- 然后再次将`doublewrite buffer`的页写入到实际的数据文件，同样的调用fsync落盘。

随后在出现需要进行页修复时，系统就可以在共享表空间内找到该页的一个副本，然后复制覆盖。

## References

《Mysql技术内幕：InnoDB存储引擎》: [https://www.amazon.cn/dp/B00ETOV48K](https://www.amazon.cn/dp/B00ETOV48K)

《File Space Management -- MySQL 8.0 Reference Manual》: [https://dev.mysql.com/doc/refman/8.0/en/innodb-file-space.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-file-space.html)

[Paper] InnoDB DoubleWrite Buffer as Read Cache using SSDs∗ : [https://www.usenix.org/legacy/events/fast12/poster_descriptions/Kangdescription2-12-12.pdf](https://www.usenix.org/legacy/events/fast12/poster_descriptions/Kangdescription2-12-12.pdf)


