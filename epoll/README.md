## epoll 源码分析

---

在 Linux 下做网络编程,一定需要对 epoll 有深入的了解,因此我在这里对 epoll 的源码做了注释,因为源码过长,所以摘出来一些重要的数据结构和关键代码,对其做了注释(从暑假拖到12月份也太真实了....)

### epoll 介绍

#### 什么是 I/O 多路复用?

在讲解 epoll 之前,我先简单介绍一下什么是多路复用,其实能想着去了解 epoll 底层实现原理的人一般不会出现这种疑问,但是我还是简单的说一下吧.

当我们在进行网络编程的时候,经常会出现一种场景.

- 我们需要从标准输入去输入数据,然后通过套接字去把数据发送出去,同时,该程序也需要通过套接字去接受对方发送过来的数据流

在单线程的情况之下,我们如果使用用 fgets 或者 scanf 等方法等待标准输入,就没办法在套接字有数据返回的时候读出数据;我们也可以使用 read 方法等待套接字有数据返回,但是这样做,就没有办法在标准输入有数据的情况下,读入数据并发送给对方.

为了解决这种问题,最简单粗暴的方式就是开多个线程,其中有线程只负责阻塞 read 套接字,有线程去阻塞在 fgets 或者 scanf 上去等待输入.

这种方式看似解决了问题,但是缺点很明显,这种多线程程序,尤其是在服务端编程的时候会导致系统开销十分巨大,线程切换占用大量时间.

I/O 多路复用技术就是解决这种问题的,我们把标准输入,套接字等都看做 I/O 的一路,多路复用的意思,就是在任何一路 I/O 有“事件”发生的情况下,通知应用程序,去处理相应的 I/O 事件,这样我们的程序就好像在单线程的情况下,在同一时刻处理多个 I/O 事件。

#### I/O 多路复用技术目前都有什么实现?

在 epoll 之前, Linux 下已经有了一些 I/O 多路复用技术的实现了,比如说 select 和 poll ,不过这两者都有一些比较大的缺陷,就比如 select 就有以下缺点

- 通过select方式单个进程能够监控的文件描述符不得超过 进程可打开的文件个数上限,默认为 1024,即便强行修改了这个上限,还会遇到性能问题.
- select轮询效率随着监控个数的增加而性能变差.
- select从内核空间返回到用户空间的是整个文件描述符数组,应用程序还需要额外再遍历整个数组才知道哪些文件描述符触发了相应事件.
- select 需要将监听的 fd 从用户空间拷贝到内核空间,然后在内核空间处理之后,在拷贝到用户空间,这里涉及到了大量的内核空间的申请内存,释放内存.
  
poll 作为 select 的加强版,虽然解决了一些 select 的问题,比如,用 poll 就可以监控文件个数超过 1024 ,但是在性能上面依旧无法满足用户越来越多的各种网络程序,因此性能更加强大的 epoll 出现了,他在内核维护了一个红黑树,通过对这颗红黑树进行操作,查找速度也比轮询的o(n)复杂度降为看o(log(n)),每次也只返回触发相应事件的文件描述符,同时也避免大量的内存申请和释放操作.

但是如果扣细节去论资排辈, epoll 在 Linux 2.5.44 才首次出现,在 2.6 中才逐渐稳定下来,可是在那个时候已经 2002 年了,隔壁 windows 在 1994 年就有了异步 I/O 大杀器 IOCP 来支持高并发网络 I/O, FreeBSD 在 2000 年就引入了 Kqueue ,而且 epoll 的性能并没有这两者高......

不过这也不怪 epoll ,由于 Linux 操作系统对 aio 的支持并不算好(期待 io_uring 能早日完善),所以 epoll 的性能自然会弱与 IOCP ,而且事实上与同是 I/O 多路复用技术的 Kqueue 之间性能差距并不大,这也就是 epoll 能如此具有生命力的一个原因.

#### 扯多了

这里主要介绍的应该是 epoll 的源码解析,告诉大家 epoll 为何如此高效,所以一切尽在代码之中,大家可以去目录下的文件中了解.

#### epoll 最新版源码(与注释版源码有出入)

[eventpoll.c](https://github.com/torvalds/linux/blob/master/fs/eventpoll.c)
[eventpoll.h](https://github.com/torvalds/linux/blob/master/include/linux/eventpoll.h)
[poll.h](https://github.com/torvalds/linux/blob/master/include/linux/poll.h)