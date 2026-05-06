> [原文地址](https://tech.meituan.com/2016/11/18/disruptor.html)，以下是文章的主要内容概述：
>
> ### 背景
>
> - Disruptor 是 LMAX 公司开发的高性能队列，用于解决内存队列延迟问题。
> - 单线程能支撑每秒 600 万订单。
> - 获得 2011 年 Oracle 官方 Duke 大奖。
>
> ### Java 内置队列问题
>
> - 介绍了 Java 内置队列及其性能问题，特别是 ArrayBlockingQueue 的加锁和伪共享问题。
>
> ### ArrayBlockingQueue 问题分析
>
> - 加锁影响性能，CAS 操作比加锁性能好。
> - 伪共享问题，通过增加变量间隔解决。
>
> ### Disruptor 设计方案
>
> - 环形数组结构，无锁设计，元素位置快速定位。
>
> ### Disruptor 实现
>
> - 单生产者和多生产者情况下的读写流程。
> - 使用 CAS 保证线程安全。
>
> ### Disruptor 性能
>
> - 提供了与 ArrayBlockingQueue 的性能对比数据，显示 Disruptor 吞吐量快 4~7 倍。
>
> ### 等待策略
>
> - 介绍了 Disruptor 中不同等待策略的适用场景。
>
> ### Log4j 2 应用场景
>
> - Log4j 2 采用 Disruptor 提高多线程并发场景下的性能。
>
> ### 性能差异
>
> - 对比了 loggers all async 和 Async Appender 的性能差异。
>
> ### 参考文档
>
> - 文章最后提供了参考文档链接。
>
> 文章详细介绍了 Disruptor 的设计原理、实现方式以及在实际应用中的性能表现，对于需要高性能队列解决方案的开发者来说，是一篇很有价值的资料。

# 高性能队列 Disruptor
