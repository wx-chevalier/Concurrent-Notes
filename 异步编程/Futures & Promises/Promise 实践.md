# Promise 实现

# 隐式与显式 Promise

# Promise Pipelining

传统 RPC 系统的缺陷之一是它们正在阻塞。 设想一个场景，您需要调用一个 API“ A”和另一个 API“ B”，然后汇总这两个调用的结果并将该结果用作另一个 API“ C”的参数。 现在，执行此操作的逻辑方法是并行调用 A 和 B，然后在完成后汇总结果并调用 C。不幸的是，在阻塞系统中，执行方法是调用 A，等待 完成它，调用 B，等待，然后聚合并调用 C。这似乎是在浪费时间，但是如果没有异步性，这是不可能的。 即使具有异步性，线性管理或扩展系统也有些困难。 幸运的是，我们有 Promise。

![Conventional RPC with Pipelining](http://dist-prog-book.com/chapter/2/images/p-2.png)

Future/Promise 可以传递，等待或链接在一起。 这些属性有助于使使用它们的程序员的生活更轻松。 这也减少了与分布式计算相关的延迟。 Promise 启用了数据流并发，这也是确定性的，并且更易于推理。

# 异常处理

# Futures and Promises in Action

## Twitter Finagle

# 链接

- http://dist-prog-book.com/chapter/2/futures.html#implicit-vs-explicit-promises
