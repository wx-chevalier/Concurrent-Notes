# Disruptor

Disruptor 是英国外汇交易公司 LMAX 开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与 I/O 操作处于同样的数量级）。基于 Disruptor 开发的系统单线程能支撑每秒 600 万订单，2010 年在 QCon 演讲后，获得了业界关注。2011 年，企业应用软件专家 Martin Fowler 专门撰写长文介绍。同年它还获得了 Oracle 官方的 Duke 大奖。

Disruptor 是一个高性能的异步处理框架，或者可以认为是最快的消息框架（轻量的 JMS），也可以认为是一个观察者模式的实现，或者事件监听模式的实现。目前，包括 Apache Storm、Camel、Log4j 2 在内的很多知名项目都应用了 Disruptor 以获取高性能。

# Links
