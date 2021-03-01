# 线程模型

在[《Linux 线程模型》](https://ng-tech.icu/Linux-Series/#/)中我们讨论过 Linux 中线程的实现以及用户态线程与内核态线程中等概念，在这里我们讨论的线程模型，则着眼于如何高效的利用多个物理核，进行工作任务的调度，使得系统能够有更高有效的吞吐，更加低的延迟。而不是被某个等待 IO 的线程阻塞或者把时间花在大量的比如系统层面的上下文切换等工作。网络 IO 中最简单的就是连接独占模型：也就是一个连接进来请求后，独占一个线程（进程）进行处理；无论其中中间在做什么事情，比如调用第三方的服务，等待过程中也是独占着整个线程。传统的如 Tomcat Servlet 就是基于这样的原理。而我们现在实际应用中常用的是 Reactor 模型，单线程 Reactor 模型譬如使用单个线程处理所有连接上的请求，使用 epoll-wait 方式，实现事件多路复用机制；典型比如 Redis，适用于简单比如小数据的内存数据的获取。每一个回调逻辑都比较简单。还有多线程 Reactor 模型，即多个线程/进程 Accept 同一个连接上的请求。

# Reactor 模型

## 单线程模型

所有的 IO 操作都在同一个 NIO 线程上面完成，NIO 线程需要处理客户端的 TCP 连接，同时读取客户端 Socket 的请求或者应答消息以及向客户端发送请求或者应答消息。如下图：

![Reactor 单线程模型](https://i.postimg.cc/cLws0kS8/1fdcd36e76359339539a507278f566d7.png)

由于采用的是非阻塞的 IO，所有 IO 操作都不会导致阻塞，从理论上来说，一个线程可以独立处理所有的 IO 相关操作，处理流程如下:

![Reactor 单线程模型时序图](https://i.postimg.cc/zfNqBwz2/65cdba67cfcee3302b88d114e2fd5baf.png)

可以看出，单线程模型只适用并发量比较小的应用场景。当一个 NIO 线程同时处理上万个连接时，处理速度会变慢，会导致大量的客户端连接超时，超时之后如果进行重发，更加会加重了 NIO 线程的负载，最终会有大量的消息积压和处理超时，NIO 线程会成为系统的瓶颈。

## 多线程模型

多线程模型与单线程模型最大的区别是有专门的一组 NIO 线程处理 IO 操作，一般使用线程池的方式实现。一个 NIO 线程同时处理多条连接，一个连接只能属于 1 个 NIO 线程，这样能够防止并发操作问题。

![Reactor 多线程模型](https://i.postimg.cc/s2JsZB1j/fbd2af5606580061718cb69254f95a71.png)

## 主从多线程模型

服务端用于接收客户端连接的不是 1 个单独的 NIO 线程了，而是采用独立的 NIO 线程池。Acceptor 接收 TCP 连接请求处理完成之后，将创建新的 SocketChannel 注册到处理连接的 IO 线程池中的某个 IO 线程上，有它去处理 IO 读写以及编解码的工作。Acceptor 只用于客户端登录、握手以及认证，一旦连接成功之后，将链路注册到线程池的 IO 线程上。

![Reactor 主从多线程模型](https://i.postimg.cc/SsNqLyzW/e774d586cd02cf2d4e7adba4b8300eac.png)

# 编程模型与协程

在进程，线程与协程的对比中我们讨论过协程的优劣，在上面讨论的多种线程模型中，我们仍然面临着一个问题，就是避免在如上回调逻辑中调用阻塞性的逻辑，否则单个事件被阻塞后整个线程组都会被阻塞。比如 Nginx 针对磁盘 IO 推出多线程支持，在 Nginx 中磁盘 IO 层面的请求，不直接由网络 IO 处理线程来进行，而是将磁盘 IO 的阶段委派给专门的单独的线程池进行；譬如使用 proxy_temp_file 从后端拉数据缓存在本地磁盘消费。

在 BIO 模型中，主要依赖多开线程的方式来提供服务端的吞吐。但是多开线程势必带来的问题就是系统层面的开销比较大（contex-switch、cache-bounding 等等），对于高性能场景典型就不太适用。如前所介绍的 Reactor 模型，我们天然地会引入多线程来解决这个问题；但是并发编程，特别是 1 个请求需要请求 N 个模块完成的情况下，如果使用异步模式，程序的复杂度就会非常复杂，并且很难于进行测试。对于异步代码的测试与正确性保障从来都是令人头疼的问题。

因此在编程模型中，我们倾向于使用类似于协程的解法；即在阻塞情况下保留当前 task 执行的上线文(栈、寄存器、signal 等)，然后切换到别的可以执行的 task 上。在 task 具备执行条件的时候将当前执行线程的 context 替换为为对应的 task 中保存的上下文，从而实现执行逻辑的切换。

- N:1 用户态线程库模型: N 个用户态下线程对应 1 个 native 的 thread，如腾讯开源的 libco 等。

- M:N 用户态线程库模型：典型如 go 语言的 goroutine、brpc 的 bthread、开源实现 libgo，并且支持 work-steal 的调度模型来避免长尾效应。

# Reactor 模式

Reactor 模型在 Linux 系统中的具体实现即是 select/poll/epoll/kqueue，像 Redis 中即是采用了 Reactor 模型实现了单进程单线程高并发。Reactor 模型的理论基础可以参考 [reactor-siemens](http://www.dre.vanderbilt.edu/%7Eschmidt/PDF/reactor-siemens.pdf)

![Redis 非阻塞与多线程模型对比](https://s2.ax1x.com/2019/11/25/MvR524.png)

Doug Lea 把 Reactor 模式分为三种类型，分别是：Basic version, Multi-threaded version 和 Multi-reactors version。

## Basic version (Single threaded)

如图所示为 Reactor 模式的单线程版本，所有的 I/O 操作都在同一个 NIO 线程上面完成。NIO 线程即是 Reactor、Acceptor 负责监听多路 socket 并 Accept 新连接，又负责分派、处理请求。对于一些小容量应用场景，可以使用单线程模型，但是对于高负载、大并发的应用却不合适。实际当中基本不会采用单线程模型。

![基础版本](https://s3.ax1x.com/2021/03/01/6P2vzn.png)

## Multi-threaded version

![Reactor 模式的多线程版本](https://s3.ax1x.com/2021/03/01/6PRCZT.png)

多线程模型中有一个专门的线程负责监听和处理所有的客户端连接，网络 IO 操作由一个 NIO 线程池负责，它包含一个任务队列和 N 个可用的线程，由这些 NIO 线程负责消息的读取、解码、编码和发送。1 个 NIO 线程可以同时处理 N 条链路，但是 1 个链路只对应 1 个 NIO 线程，防止发生并发操作问题。

在绝大多数场景下，Reactor 多线程模型都可以满足性能需求。但是在更高并发连接的场景（如百万客户端并发连接），它的性能似乎不能满足需求，于是便出现了下面的多 Reactor（主从 Reactor）模型。

## Multi-reactors version

![主从 Reactor 模型](https://s3.ax1x.com/2021/03/01/6PRAJJ.png)

这种模型是将 Reactor 分成两部分，mainReactor 负责监听 server socket、accept 新连接，并将建立的 socket 分派给 subReactor；subReactor 负责多路分离已连接的 socket，读写网络数据；而对业务处理的功能，交给 worker 线程池来完成。

# 案例：Redis Reactor

## 核心组件

![](http://www.dengshenyu.com/assets/redis-reactor/reactor-mode3.png)

- Handles：表示操作系统管理的资源，我们可以理解为 fd。

- Synchronous Event Demultiplexer：同步事件分离器，阻塞等待 Handles 中的事件发生。

- Initiation Dispatcher：初始分派器，作用为添加 Event handler(事件处理器)、删除 Event handler 以及分派事件给 Event handler。也就是说，Synchronous Event Demultiplexer 负责等待新事件发生，事件发生时通知 Initiation Dispatcher，然后 Initiation Dispatcher 调用 event handler 处理事件。

- Event Handler：事件处理器的接口

- Concrete Event Handler：事件处理器的实际实现，而且绑定了一个 Handle。因为在实际情况中，我们往往不止一种事件处理器，因此这里将事件处理器接口和实现分开，与 C++、Java 这些高级语言中的多态类似。

## 处理逻辑

Reactor 模型的基本的处理逻辑为：(1)我们注册 Concrete Event Handler 到 Initiation Dispatcher 中。(2)Initiation Dispatcher 调用每个 Event Handler 的 get_handle 接口获取其绑定的 Handle。(3)Initiation Dispatcher 调用 handle_events 开始事件处理循环。在这里，Initiation Dispatcher 会将步骤 2 获取的所有 Handle 都收集起来，使用 Synchronous Event Demultiplexer 来等待这些 Handle 的事件发生。(4)当某个(或某几个)Handle 的事件发生时，Synchronous Event Demultiplexer 通知 Initiation Dispatcher。(5)Initiation Dispatcher 根据发生事件的 Handle 找出所对应的 Handler。(6)Initiation Dispatcher 调用 Handler 的 handle_event 方法处理事件。

时序图如下：

![](http://www.dengshenyu.com/assets/redis-reactor/reactor-mode4.png)

抽象来说，Reactor 有 4 个核心的操作：

1. add 添加 socket 监听到 reactor，可以是 listen socket 也可以使客户端 socket，也可以是管道、eventfd、信号等
2. set 修改事件监听，可以设置监听的类型，如可读、可写。可读很好理解，对于 listen socket 就是有新客户端连接到来了需要 accept。对于客户端连接就是收到数据，需要 recv。可写事件比较难理解一些。一个 SOCKET 是有缓存区的，如果要向客户端连接发送 2M 的数据，一次性是发不出去的，操作系统默认 TCP 缓存区只有 256K。一次性只能发 256K，缓存区满了之后 send 就会返回 EAGAIN 错误。这时候就要监听可写事件，在纯异步的编程中，必须去监听可写才能保证 send 操作是完全非阻塞的。
3. del 从 reactor 中移除，不再监听事件
4. callback 就是事件发生后对应的处理逻辑，一般在 add/set 时制定。C 语言用函数指针实现，JS 可以用匿名函数，PHP 可以用匿名函数、对象方法数组、字符串函数名。

Reactor 只是一个事件发生器，实际对 socket 句柄的操作，如 connect/accept、send/recv、close 是在 callback 中完成的。具体编码可参考下面的 Swoole 伪代码：

![](http://rango.swoole.com/static/io/6.png)

Reactor 模型还可以与多进程、多线程结合起来用，既实现异步非阻塞 IO，又利用到多核。目前流行的异步服务器程序都是这样的方式：如

- Nginx：多进程 Reactor
- Nginx+Lua：多进程 Reactor+协程
- Golang：单线程 Reactor+多线程协程
- Swoole：多线程 Reactor+多进程 Worker

协程从底层技术角度看实际上还是异步 IO Reactor 模型，应用层自行实现了任务调度，借助 Reactor 切换各个当前执行的用户态线程，但用户代码中完全感知不到 Reactor 的存在。

# TBD

- https://blog.csdn.net/u013074465/article/details/46276967
