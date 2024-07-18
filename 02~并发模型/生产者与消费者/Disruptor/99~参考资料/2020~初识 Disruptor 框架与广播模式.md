> [原文地址](https://github.com/yuanmabiji/Java-SourceCode-Blogs/blob/master/Disruptor/%E5%88%9D%E8%AF%86Disruptor%E6%A1%86%E6%9E%B6.md)
> 以下是该文档的主要内容概述：
>
> ### 1. 前言
>
> - **LMAX 交易平台**：运行在 JVM 平台上的零售金融交易平台，单个线程每秒处理 6 百万订单。
> - **Disruptor 框架**：支持高并发性能，2011 年获得 Duke’s 程序框架创新奖。
> - **核心数据结构**：Ring Buffer（环形数组），实现无锁的并发操作。
>
> ### 2. Disruptor 框架特点
>
> 1. **预加载内存**：使用内存池。
> 2. **无锁化**：避免锁机制，提高并发性能。
> 3. **单线程写**：每个生产者独立写入数据。
> 4. **消除伪共享**：通过填充缓存行来减少缓存一致性开销。
> 5. **使用内存屏障**：确保内存操作的顺序性。
> 6. **序号栅栏机制**：协调生产者和消费者之间的数据交换进度。
>
> ### 3. 相关概念
>
> - **Disruptor**：核心类，持有 RingBuffer、消费者线程池等。
> - **Ring Buffer**：环形数组，生产者和消费者之间交换数据的桥梁。
> - **Sequencer**：实现并发算法，传递数据。
> - **Sequence**：标识 Ring Buffer 和消费者 Event Processor 的处理进度。
> - **Sequence Barrier**：协调生产者和消费者之间的数据交换进度。
> - **Wait Strategy**：决定消费者如何等待生产者的策略。
> - **Event Processor**：消费者线程，从 Ring Buffer 获取数据。
> - **Event Handler**：实现业务逻辑的 Handler。
> - **Producer**：生产数据的类。
>
> ### 4. 入门 DEMO
>
> - **LongEvent**：定义事件数据。
> - **LongEventFactory**：创建事件实例。
> - **LongEventHandler**：处理事件的逻辑。
> - **LongEventTranslatorOneArg**：将数据翻译到事件中。
> - **LongEventMain**：演示如何使用 Disruptor 框架。
>
> 示例代码展示了如何创建 Disruptor 实例，处理事件，并使用 RingBuffer 发布事件。

# 初识 Disruptor 框架与广播模式
