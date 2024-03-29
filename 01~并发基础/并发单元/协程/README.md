# 进程，线程与协程

在未配置 OS 的系统中，程序的执行方式是顺序执行，即必须在一个程序执行完后，才允许另一个程序执行；在多道程序环境下，则允许多个程序并发执行。程序的这两种执行方式间有着显著的不同。也正是程序并发执行时的这种特征，才导致了在操作系统中引入进程的概念。**进程是资源分配的基本单位，线程是资源调度的基本单位**。

应用启动体现的就是静态指令加载进内存，进而进入 CPU 运算，操作系统在内存开辟了一段栈内存用来存放指令和变量值，从而形成了进程。早期的操作系统基于进程来调度 CPU，不同进程间是不共享内存空间的，所以进程要做任务切换就要切换内存映射地址。由于进程的上下文关联的变量，引用，计数器等现场数据占用了打段的内存空间，所以频繁切换进程需要整理一大段内存空间来保存未执行完的进程现场，等下次轮到 CPU 时间片再恢复现场进行运算。

这样既耗费时间又浪费空间，所以我们才要研究多线程。一个进程创建的所有线程，都是共享一个内存空间的，所以线程做任务切换成本就很低了。现代的操作系统都基于更轻量的线程来调度，现在我们提到的“任务切换”都是指“线程切换”。

为了避免冗余，关于**进程与线程**的讨论请参考 [《Linux 与操作系统/进程管理》](https://ngte-infras.gitbook.io?q=进程管理)章节。

# Coroutine | 协程

协程是用户模式下的轻量级线程，最准确的名字应该叫用户空间线程（User Space Thread），在不同的领域中也有不同的叫法，譬如纤程(Fiber)、绿色线程(Green Thread)等等。操作系统内核对协程一无所知，协程的调度完全有应用程序来控制，操作系统不管这部分的调度；一个线程可以包含一个或多个协程，协程拥有自己的寄存器上下文和栈，协程调度切换时，将寄存器上细纹和栈保存起来，在切换回来时恢复先前保运的寄存上下文和栈。

协程的优势如下：

- 节省内存，每个线程需要分配一段栈内存，以及内核里的一些资源
- 节省分配线程的开销（创建和销毁线程要各做一次 syscall）
- 节省大量线程切换带来的开销
- 与 NIO 配合实现非阻塞的编程，提高系统的吞吐

比如 Golang 里的 go 关键字其实就是负责开启一个 Fiber，让 func 逻辑跑在上面。而这一切都是发生的用户态上，没有发生在内核态上，也就是说没有 ContextSwitch 上的开销。协程的实现库中笔者较为常用的譬如 Go Routine、[node-fibers](https://github.com/laverdet/node-fibers)、[Java-Quasar](https://github.com/puniverse/quasar) 等。

## 协程切换实验

用户态线程的优势，切换 50 ～ 100ns(2GhzCPU 情况下 100 ～ 200 个 cycle)级别 相比 linux 原生内核线程切换 1 ～ 2 us。近 1 个数量级的性能提升。

````s
// 以下简单测试了 intel E5-2670 v3 CPU (4.9.65 内核情况下)在各种switch方面的性能和参考文献的一些性能数据。
// 参考文献(https://blog.tsunanet.net/2010/11/how-long-does-it-take-to-make-context.html)
// https://github.com/tsuna/contextswitch/blob/master/timesyscall.c
* 陷入内核的context-switch（使用轻量级的gettid调用进行测试）
Intel 5150: 105ns/syscall
Intel E5440: 87ns/syscall
Intel E5520: 58ns/syscall
Intel X5550: 52ns/syscall
Intel L5630: 58ns/syscall
Intel E5-2620: 67ns/syscall
Intel E5-2670 v3：211ns/syscall
​```

也就是从当前看，纯粹从陷入内核方面已经不太跟常规切换的context一样耗时间，基本在百来个cpu cycle能够完成。

```s
* 进程/线程之间的context-switch（产生竞争情况下使用SYS_futex陷入内核等待的性能，2个进程相互唤醒的方式测试）
// https://github.com/tsuna/contextswitch/blob/master/timectxsw.c
// https://github.com/tsuna/contextswitch/blob/master/timetctxsw.c
Intel 5150: ~4300ns/context switch
Intel E5440: ~3600ns/context switch
Intel E5520: ~4500ns/context switch
Intel X5550: ~3000ns/context switch
Intel L5630: ~3000ns/context switch
Intel E5-2620: ~3000ns/context switch
Intel E5-2670 v3: ~1769.5ns/context switch
````

# Go 的协程模型

Go 线程模型属于多对多线程模型，在操作系统提供的内核线程之上，Go 搭建了一个特有的两级线程模型。Go 中使用使用 Go 语句创建的 Goroutine 可以认为是轻量级的用户线程，Go 线程模型包含三个概念：

- G: 表示 Goroutine，每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，可重用。G 并非执行体，每个 G 需要绑定到 P 才能被调度执行。

- P: Processor，表示逻辑处理器，对 G 来说，P 相当于 CPU 核，G 只有绑定到 P(在 P 的 local runq 中)才能被调度。对 M 来说，P 提供了相关的执行环境（Context），如内存分配状态（mcache），任务队列（G）等，P 的数量决定了系统内最大可并行的 G 的数量（物理 CPU 核数 >= P 的数量），P 的数量由用户设置的 GOMAXPROCS 决定，但是不论 GOMAXPROCS 设置为多大，P 的数量最大为 256。

- M: Machine，OS 线程抽象，代表着真正执行计算的资源，在绑定有效的 P 后，进入 schedule 循环；M 的数量是不定的，由 Go Runtime 调整，为了防止创建过多 OS 线程导致系统调度不过来，目前默认最大限制为 10000 个。

在 Go 中每个逻辑处理器(P)会绑定到某一个内核线程上，每个逻辑处理器（P）内有一个本地队列，用来存放 Go 运行时分配的 goroutine。多对多线程模型中是操作系统调度线程在物理 CPU 上运行，在 Go 中则是 Go 的运行时调度 Goroutine 在逻辑处理器（P）上运行。

![Go 语言的线程实现模型](https://assets.ng-tech.icu/item/20230524154256.png)

Go 的栈是动态分配大小的，随着存储数据的数量而增长和收缩。每个新建的 Goroutine 只有大约 4KB 的栈。每个栈只有 4KB，那么在一个 1GB 的 RAM 上，我们就可以有 256 万个 Goroutine 了，相对于 Java 中每个线程的 1MB，这是巨大的提升。Golang 实现了自己的调度器，允许众多的 Goroutines 运行在相同的 OS 线程上。就算 Go 会运行与内核相同的上下文切换，但是它能够避免切换至 ring-0 以运行内核，然后再切换回来，这样就会节省大量的时间。

在 Go 中存在两级调度:

- 一级是操作系统的调度系统，该调度系统调度逻辑处理器占用 cpu 时间片运行；
- 一级是 Go 的运行时调度系统，该调度系统调度某个 Goroutine 在逻辑处理上运行。

使用 Go 语句创建一个 Goroutine 后，创建的 Goroutine 会被放入 Go 运行时调度器的全局运行队列中，然后 Go 运行时调度器会把全局队列中的 Goroutine 分配给不同的逻辑处理器（P），分配的 Goroutine 会被放到逻辑处理器（P)的本地队列中，当本地队列中某个 Goroutine 就绪后待分配到时间片后就可以在逻辑处理器上运行了。

![](https://assets.ng-tech.icu/item/20230524144622.png)

# Java 协程的讨论

目前，JVM 本身并未提供协程的实现库，像 Quasar 这样的协程框架似乎也仍非主流的并发问题解决方案，在本部分我们就讨论下在 Java 中是否有必要一定要引入协程。在普通的 Web 服务器场景下，譬如 Spring Boot 中默认的 Worker 线程池线程数在 200（50 ~ 500）左右，如果从线程的内存占用角度来考虑，每个线程上下文约 128KB，那么 500 个线程本身的内存占用在 60M，相较于整个堆栈不过尔尔。而 Java 本身提供的线程池，对于线程的创建与销毁都有非常好的支持；即使 Vert.x 或 Kotlin 中提供的协程，往往也是基于原生线程池实现的。

从线程的切换开销的角度来看，我们常说的切换开销往往是针对于活跃线程；而普通的 Web 服务器天然会有大量的线程因为请求读写、DB 读写这样的操作而挂起，实际只有数十个并发活跃线程会参与到 OS 的线程切换调度。而如果真的存在着大量活跃线程的场景，Java 生态圈中也存在了 Akka 这样的 Actor 并发模型框架，它能够感知线程何时能够执行工作，在用户空间中构建运行时调度器，从而支持百万级别的 Actor 并发。

实际上我们引入协程的场景，更多的是面对所谓百万级别连接的处理，典型的就是 IM 服务器，可能需要同时处理大量空闲的链接。此时在 Java 生态圈中，我们可以使用 Netty 去进行处理，其基于 NIO 与 Worker Thread 实现的调度机制就很类似于协程，可以解决绝大部分因为 IO 的等待造成资源浪费的问题。而从并发模型对比的角度，如果我们希望能遵循 Go 中以消息传递方式实现内存共享的理念，那么也可以采用 Disruptor 这样的模型。

# Links

- [谈谈协程](http://mp.weixin.qq.com/s/966vBRDHNY0taIWEseQo6w)

- [微服务的各种线程模型及其权衡](http://www.infoq.com/cn/articles/engstrand-microservice-threading)
