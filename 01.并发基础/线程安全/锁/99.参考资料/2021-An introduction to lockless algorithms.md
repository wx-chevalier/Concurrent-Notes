> [原文地址](https://lwn.net/Articles/844224/)

# An introduction to lockless algorithms

Linux kernel 里面有些场景无法使用普通的 locking primitives（互斥锁接口），或者担心这些互斥锁带来的性能影响，这种时候就可以考虑 lockless algorithm 了，也就是无锁算法。因此 lockless 会时不时地出现在 LWN 上。最后一次被提及是在去年 7 月，也因此促使我撰写了这一系列文章。而更加频繁被提到的话题则是 read-copy-update（也就是 RCU，2007 年就有好多 LWN 文章介绍了）、引用计数（reference counting）、以及各种将 lockess primitives 包装成更高级别更容易理解的 API 的方法。这些文章会深入探讨 lockless 算法背后的概念，以及如何在内核中使用。

关于 memory model 的底层知识是大家公认的进阶内容了，即使是最老练的内核开发者也会感到发憷。编者在 7 月份的文章中写到，"需要一种特殊的大脑才能真正理解 memory model"。有人说，Linux 内核的内存模型（尤其是 Documentation/memory-barriers.txt）可以用来吓唬小孩子，光是 "acquire"（获取）和 "release"（释放）这两个词就让人很头痛了。

同时，像 RCU 和 seqlocks 这样的机制在内核中被广泛使用，几乎每个开发者迟早都会遇到基本的 lockless 编程接口。出于这个原因，大家最好至少让自己对 lockless 的基本函数有个简单的了解。在本系列中，我将解释 acquire 和 release 的真正含义，并介绍五种相对简单的模式，它们可以涵盖 lockless 接口在大多数场景下的应用。

# Acquire, release, and "happens before"

为了从相对来说简单易懂的同步接口（synchronization primitives）转向 lockless 编程方式，我们首先要了解为什么加锁会是有必要的。通常，谈到锁的价值的时候，一般都是在讲互斥的情况（mutual exclusive）：加锁可以防止多个同时执行的线程对同一个数据进行并发地读或写操作。但 "concurrently（并发）"到底是什么意思呢？当线程 T 处理完某个数据之后，线程 U 开始使用该数据，这有什么问题吗？

为了回答这些问题，Leslie Lamport 在 1978 年的论文 "时间、时钟和分布式系统中的事件排序（Time, Clocks and the Ordering of Events in a Distributed System" 中建立的一个理论框架就有用武之地了。根据该论文，分布式系统中的事件(event) 的顺序可以根据一个事件 P 是否发生在另一个事件 Q 之前定义：

- 同一个线程内的事件都是按顺序发生的，也称之为"total ordering(完全顺序)"。通俗地讲，对于同一线程的任意两个事件，你总能说出哪个先发生，哪个后发生。
- 如果两个时间不是在同一个线程内的，那么如果事件 P 是在做 message send，而事件 Q 是与之相对应的 message receive，那么事件 P 就一定发生在事件 Q 之前。
- 这种关系是具有传递性的。因此，如果事件 P 发生在事件 Q 之前，而事件 Q 发生在事件 R 之前，那么事件 P 也会发生在 R 之前。

"在其之前发生" （happen before）的这种关系是 partial ordering（不完全顺序），也就是说有可能有两个事件 P 和 Q，我们无法判断这两个事件是谁先谁后的。当这种情况发生时，就意味着这两个事件是 concurrent 的(并发的)。还记得加锁是如何预防对同一数据结构的并发访问吗？这是因为，当你用锁保护一个数据结构时，所有对该数据结构的访问都会形成一个 total ordering（完全顺序关系），就像它们都是来自同一个线程的操作一样。Lamport 的论文还提供了一个基本概念，即当锁被从一个线程交给另一个线程时会发生什么：会有一些 "message passing" 机制来确保线程 T 的 unlock 操作是发生在线程 U 的 lock 操作之前的。

事实证明，这种说法并不仅仅是一个理论而已：CPU 为了确保它们的 cache 一致性，会需要通过总线（如 Intel 的 QPI 或 AMD 的 HyperTransport）来传递 message。然而，本文中我们不需要深入到这些细节中。所以，我们这里将 "happen before" 的定义推广为所有同步原语（synchronization primitive）的一个公共定义。

Lamport 的基本观点是，当两个线程对同一数据结构进行对称操作时，就需要 synchronization。在我们推广之后的说明中，我们会列举出那些使一个线程与另一个线程 synchronize(同步)的配对操作（例如通过同一个队列来发送和接收消息）。此外，我们将把每一对操作中都分为 release（释放）或 acquire（获取）两个操作。也就是说我们后面只会用"释放了"或者"获取了"这种语义来说明。

在每一对操作内，具有释放语义的操作与相应的获取语义操作是同步操作。"同步(synchronizing) "意味着每当一个线程内执行释放操作、另一个线程执行相应的获取操作时，就表示这个进行释放的线程是发生在获取线程之前的。"happen before" 一般来说仍然是一个不完全顺序关系（partial order），但现在它可以跨越两个线程了，甚至是更多线程，这要归功于传递性。更正式的说法是：

- 在单个线程中，各个操作都是完全顺序的（total ordering）
- 如果一个进行释放动作的操作 P 与一个进行获取动作的操作 Q 是同步的（synchronized），就意味着操作 P 发生在操作 Q 之前，哪怕这两个操作是发生在不同的线程中的。
- 像之前一样，这个关系是具有传递性的，也具有 partial ordering 的特性

按照旧的定义，message send 是属于释放操作，而 message receive 属于获取操作。message send 和 message receive 就使得发送消息的线程与接收消息的线程（或多个接收线程）实现了同步。我们也可以根据新定义来重新解释我们之前的观点：为了让锁有效果，unlock 操作就是释放操作，并且必须与 lock 操作进行同步，加锁操作就是获取操作。无论是否有人竞争这把锁，最终结果都保证了一个 thread 和另一个 thread 中的操作有了"happen before"的关系。

获取操作和释放操作似乎是一个抽象的概念，但它们确实为许多常见的多线程编程实例进行了简单的解释。例如，如果有两个用户空间线程都需要访问一个全局变量 s：
