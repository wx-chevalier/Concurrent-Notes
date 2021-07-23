# 线程池

线程池是一种使用户可以访问可以进行工作的一组就绪的空闲线程的抽象。线程池的实现会照顾到工作线程的创建，管理和调度，如果不小心处理，它们很容易变得棘手且代价高昂。线程池具有许多不同的风格，具有许多用于调度和执行任务的技术，并且具有固定数量的线程，或者具有根据负载动态调整自身大小的能力。Java 的 Executor 就是典型的线程池的实现，Executor 包含了一系列 Runnable 的任务，它对上屏蔽了任务具体执行细节的抽象。这些详细信息（例如选择运行任务的线程，任务的计划方式）由 Executor 接口的基础实现管理。

类似于 Executor，Scala 在 `scala.concurrent` 包中提供了 ExecutionContext 类，其基础的特性类似于 Java 的 Executor；它负责并发高效地执行计算，而无需池中的用户去担心诸如调度之类的事情。更重要地是，ExecutionContext 同样可被当做接口，我们能够改变线程池地底层实现而保证上层接口的一致性。在诸多的实现中，Scala 的默认 ExecutionContext 的实现是基于 Java 的 ForkJoinPool，一种具有 work-stealing 算法的线程池实现，其中空闲线程拾取先前安排给其他繁忙线程的任务。ForkJoinPool 是一种流行的线程池实现，因为它比 Executors 的性能有所提高，能够更好地避免池引起的死锁，并最大程度地减少了在线程之间切换所花费的时间。

# Links

- https://mp.weixin.qq.com/s/skBA9RwVBLnw8BYZhcUSrA Java 线程池原理及最佳实践（1.5W 字，面试必问）
