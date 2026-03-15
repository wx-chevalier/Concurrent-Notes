# Reactor

在[《Linux 线程模型》](https://ng-tech.icu/books/Linux-Notes/#/)中我们讨论过 Linux 中线程的实现以及用户态线程与内核态线程中等概念，在这里我们讨论的线程模型，则着眼于如何高效的利用多个物理核，进行工作任务的调度，使得系统能够有更高有效的吞吐，更加低的延迟。而不是被某个等待 IO 的线程阻塞或者把时间花在大量的比如系统层面的上下文切换等工作。

网络 IO 中最简单的就是连接独占模型：也就是一个连接进来请求后，独占一个线程（进程）进行处理；无论其中中间在做什么事情，比如调用第三方的服务，等待过程中也是独占着整个线程。或者依赖多开线程的方式来提供服务端的吞吐。但是多开线程势必带来的问题就是系统层面的开销比较大（contex-switch、cache-bounding 等等），对于高性能场景典型就不太适用。传统的如 Tomcat Servlet 就是基于这样的原理，而我们现在实际应用中常用的是 Reactor 模型。Doug Lea 把 Reactor 模式分为三种类型，分别是：Basic version, Multi-threaded version 和 Multi-reactors version。

> 实践案例可以参阅《[Database-Notes/Redis](https://github.com/wx-chevalier/Database-Notes?q=)》等相关章节。

单线程 Reactor 模型譬如使用单个线程处理所有连接上的请求，使用 epoll-wait 方式，实现事件多路复用机制；典型比如 Redis，适用于简单比如小数据的内存数据的获取，每一个回调逻辑都比较简单。还有多线程 Reactor 模型，即多个线程/进程 Accept 同一个连接上的请求。Reactor 模型在 Linux 系统中的具体实现即是 select/poll/epoll/kqueue，像 Redis 中即是采用了 Reactor 模型实现了单进程单线程高并发。

# Links

- https://cubox.pro/c/mMiamX 彻底搞懂事件驱动模型 - Reactor
