# Actor

今天当人们谈论消息传递时，它们主要是指 Actor 模型。它是自 1970 年以来一直在发展的一种普遍存在的通用消息传递编程模型，如今已用于构建大规模可伸缩系统。

20 世纪 70 年代 Carl Hewitt 首次提出了 Actor 模型，而后 Erlang 为布道 Actor 做了最大的贡献，比如 Erlang 的创始人 Joe Armstrong 也是“任其崩溃”哲学的先驱。Actor 模型可以看做面向对象模型在并发编程领域的扩展，其精心设计了消息传输和封装的机制，强调了面向对象的精髓。Actor 模型是一个概念模型，用于处理并发计算。

Actor 是分布式存在的内存状态及单线程计算单元，一个 ID 对应的 Actor 只会在集群种存在一个（有状态的 Actor 在集群中一个 ID 只会存在一个实例，无状态的可配置为根据流量存在多个），使用者只需要通过 ID 就能随时访问不需要关注该 Actor 在集群的什么位置。单线程计算单元保证了消息的顺序到达，不存在 Actor 内部状态竞用问题。

尽管 actor 模型的起源可追溯到 1970 年代，但正如该领域最近发表的论文和系统所展示的那样，它仍在开发中，并已被纳入当今的编程语言中。有一些健壮的工业实力参与者系统正在为大型可扩展分布式系统提供动力。例如，Akka 已用于为 PayPal 的数十亿笔交易提供服务（（Sucharitakul，2016 年）。Erlang 已用于为 WhatsApp 的亿万用户发送消息，（Reed，2012 年），而 Orleans 已用于为 Halo 4 的数百万玩家提供服务 。（McCaffrey，2015 年）围绕监视，处理容错和管理参与者生命周期，有几种不同的方法来构建工业参与者框架。

# 最初的 Actor 模型

![](https://i.postimg.cc/9MgfnSTJ/image.png)

Actor 由 3 部分组成：状态（State）+行为（Behavior）+邮箱（Mailbox），State 是指 Actor 对象的变量信息，存在于 Actor 之中，Actor 之间不共享内存数据，Actor 只会在接收到消息后，调用自己的方法改变自己的 state，从而避免并发条件下的死锁等问题；Behavior 是指 Actor 的计算行为逻辑；邮箱建立 Actor 之间的联系，一个 Actor 发送消息后，接收消息的 Actor 将消息放入邮箱中等待处理，邮箱内部通过队列实现，消息传递通过异步方式进行。

![](https://tva1.sinaimg.cn/large/007DFXDhgy1g44cph1xu1j30hc0bwq3c.jpg)

# TBD

- http://dist-prog-book.com/chapter/3/message-passing.html#introduction
