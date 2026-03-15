# 并发模型

并发编程有多种常见模型，其中最主要的包括多线程和消息传递。从理论角度来看，这两种模型是等价的。多线程模型因为能够自然地对应到多核处理器上，所以得到了主流操作系统的广泛支持。同时，多线程的概念也相对直观，因此被许多主流编程语言采纳为语言特性或扩展库。相比之下，基于消息的并发编程模型在主流编程语言中的支持较少。Erlang 语言是支持基于消息传递并发模型的典型代表，它的并发单元之间不共享内存。而 Go 语言则可以说是基于消息并发模型的集大成者，它将基于 CSP（通信顺序进程）模型的并发编程内置到了语言中。通过简单的 go 关键字，就能轻松启动一个 Goroutine（Go 语言的轻量级线程）。与 Erlang 不同的是，Go 语言的 Goroutine 之间是共享内存的。

以下是几种常见的并发模型：

- 线程与锁模型：尽管这种模型有一些众所周知的缺点，但它仍然是其他模型的技术基础，也是许多并发软件开发的首选方案。它的核心思想是使用多个线程并通过锁机制来控制共享资源的访问。

- 函数式编程模型：函数式编程正变得越来越重要，其中一个重要原因是它为并发编程和并行编程提供了出色的支持。由于函数式编程消除了可变状态，所以从根本上保证了线程安全，并且易于并行执行。例如，Haskell 语言就是一个纯函数式编程语言，它的并发编程模型非常强大。

- Clojure 的分离标识与状态模型：Clojure 是一种将指令式编程和函数式编程巧妙结合的语言。它通过分离标识和状态，在两种编程范式之间取得了平衡，从而发挥了两者的优势。这种方法使得并发编程变得更加简单和安全。

- Actor 模型：这是一种适用范围很广的并发编程模型。它不仅适用于共享内存系统，也适用于分布式内存系统，甚至可以解决地理分布式的问题。Actor 模型还能提供强大的容错能力。在这个模型中，每个 actor 是一个独立的计算单元，通过消息传递进行通信。

- 通信顺序进程（CSP）模型：表面上看，CSP 模型与 Actor 模型很相似，两者都基于消息传递的概念。但是，CSP 模型更加强调用于传递信息的通道，而 Actor 模型则更关注通道两端的实体。使用 CSP 模型编写的代码通常会呈现出明显不同的风格。Go 语言就是基于 CSP 模型的典型代表。

- 数据级并行模型：现代的笔记本电脑中都隐藏着一台超级计算机——GPU（图形处理器）。GPU 利用了数据级并行的特性，不仅可以快速进行图像处理，还可以应用于更广泛的领域。例如，如果需要进行有限元分析、流体力学计算或其他大规模数值计算，GPU 的性能将是最佳选择。CUDA 和 OpenCL 是两种常用的 GPU 并行计算框架。

- Lambda 架构：随着大数据时代的到来，并行计算变得越来越重要。现在，我们只需要增加计算资源，就能够处理 TB 级别的数据。Lambda 架构结合了 MapReduce 和流式处理的特点，是一种能够处理多种大数据问题的架构。它将数据处理分为批处理层和速度层，能够同时满足大规模数据处理和实时数据处理的需求。

以下是包含 Java 在内的主要编程语言及其并发编程特性的更新列表：

1. Java:

   - 提供 java.util.concurrent 包，包含丰富的并发工具。
   - 支持线程、同步原语（synchronized, volatile）。
   - 提供 ExecutorService 用于线程池管理。
   - 支持 Fork/Join 框架用于并行处理。
   - 提供 CompletableFuture 用于异步编程。
   - 支持并发集合如 ConcurrentHashMap。

2. C++:

   - 从 C++11 开始，引入了线程库。
   - 提供 std::thread, std::mutex, std::condition_variable 等。
   - 支持原子操作和内存模型。

3. Python:

   - 提供 threading 和 multiprocessing 模块。
   - asyncio 库支持异步编程模式。
   - queue 模块支持生产者-消费者模式。
   - 全局解释器锁（GIL）限制了多线程性能。

4. Go:

   - 内置 goroutines 和 channels，设计用于并发。
   - sync 包提供互斥锁、读写锁等同步原语。
   - 支持 select 语句用于多路复用。

5. Rust:

   - 所有权系统和借用检查器提供并发安全保证。
   - std::sync 和 std::thread 模块支持并发编程。
   - 支持无数据竞争的并发。

6. JavaScript (Node.js):

   - 基于事件循环和回调的异步模型。
   - Promise 和 async/await 支持异步编程。
   - Worker Threads 支持多线程处理。

7. C#:

   - Task Parallel Library (TPL) 提供高级并发支持。
   - async/await 关键字支持异步编程。
   - 提供线程安全集合和并发模式实现。

8. Scala:

   - Akka 框架提供强大的并发和分布式编程模型。
   - 支持 Actor 模型。
   - 与 Java 并发库兼容。

9. Erlang/Elixir:

   - 基于 Actor 模型，设计用于并发和分布式系统。
   - 轻量级进程和消息传递。

10. Kotlin:
    - 协程（Coroutines）提供轻量级并发。
    - 与 Java 并发库完全兼容。
    - 提供 Flow API 用于异步数据流。

# Links

- https://medium.com/one-datum-at-a-time/an-exploration-into-concurrent-programming-and-concurrency-models-b22144950a88

- [concurrency-models](http://tutorials.jenkov.com/java-concurrency/concurrency-models.html)

- [七周七并发模型](https://drive.wps.cn/view/l/3db758274cf94555a456332436ec5f19)

- [Lock-Free and Wait-Free, definition and examples](http://concurrencyfreaks.blogspot.co.id/2013/05/lock-free-and-wait-free-definition-and.html)

- [并行编程：并发级别](http://www.cnblogs.com/jiayy/p/3246167.html)
