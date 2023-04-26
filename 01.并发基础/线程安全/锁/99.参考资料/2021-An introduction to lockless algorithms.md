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

```sh
    thread 1                              thread 2
    --------------------------------      ------------------------
    s = "hello";
    pthread_create(&t, NULL, t2, NULL);
                                          puts(s);
                                          s = "world";
    pthread_join(t, NULL);
    puts(s);
```

那么这里对变量的访问是否安全？是否能确定 thread 2 从 s 中读出的内容是 "hello"，并且 thread 1 在 pthread_join()后输出的 s 的值一定为 "world"？答案是可以确定，下面我们用获取和释放的语义来解释原因：

- pthread_create() 具有释放的语义，并与 thread 2（具有获取语义）的 start（即线程开始执行）有同步关系。因此，在线程创建之前所写的任何数据都可以安全地从线程中访问到。
- thread 2 的 exit 操作具有释放语义，并与 pthread_join() （其具有获取语义）有同步关系。因此，线程在退出之前写的任何东西都可以在 pthread_join()之后安全地访问到。

请注意，这里成功地在不用 lock 的情况下将数据从一个线程可靠地传递到了另一个线程。恭喜，你已经完成了 lockless 编程的第一个例子。总结一下：

- 如果程序作者想让 thread 2 "看到 " thread 1 到目前为止发生的所有事情的效果，那么两个线程需要相互同步，也就是需要在 thread 1 中进行释放操作，在 thread 2 中进行获取操作。

- 只要知道哪些 API 提供了 acquire/release 语义，就可以利用这些 API 来写出确保 order 的代码。

在了解了释放和获取语义对这种 high-level synchronization primitive （高层次的同步原语）是如何起作用之后，我们现在可以再看一下对同一个内存数据访问中可以如何利用它们了。

# The message-passing pattern

在前一段中，我们已经看到了 pthread_create()和 pthread_join()的获取和释放语义是如何让线程的创建者与该线程进行信息交换的，以及如何反向传递信息。现在，我们将看到在线程运行过程中如何采用 lockless 方式进行同步。如果传递的 message 只是一个简单的标量值，例如就是一个布尔值，那么可以通过对这个内存地址进行读写就访问到了。然而，如果消息是一个指针会怎样呢？

```sh
    thread 1                            thread 2
    --------------------------------    ------------------------
    a.x = 1;
    message = &a;                       datum = message;
                                        if (datum != NULL)
                                          printk("%d\n", datum->x);
```

最初这个 message 是 NULL 的话，那么我们不清楚 thread 2 打印出来的是 NULL 还是 &a。问题是，哪怕这里 “datum = message” 操作读到的是 &a，这个赋值操作其实与 thread 1 中的赋值操作“message = &a ”并无同步关系。因此，这两个线程并没有一个"happens before"的关系。

```sh
    a.x = 1;                            datum = message;
       |                                    |
       |   happens before                   |
       v                                    v
    message = &a;                       datum->x
```

因为这两个线程的执行并没有同步关系，所以不能保证 datum->x 读出来的值肯定是 1，毕竟我们不知道对 a.x 的赋值是否发生在读取 datum->x 之前。要确保 datum-> 读出来的值必定是 1 的话，就需要让 store 和 load 操作拥有释放和获取的语义。

为此，我们定义了 "store-release" 和 "load-acquire" 这两种操作。store-release 操作 （取名为操作 P）除了会向内存位置写入数据之外，也能确保与采用 load-acquire 方式读取这个内存数据的操作（名为操作 Q）是互相同步的。因此我们可以使用 Linux 的 smp_store_release()和 smp_load_acquire()来修复上述代码：

```sh
    thread 1                                  thread 2
    --------------------------------          ------------------------
    a.x = 1;
    smp_store_release(&message, &a);          datum = smp_load_acquire(&message);
                                              if (datum != NULL)
                                                printk("%x\n", datum->x);
```

这样改动之后，如果 datum 是 &a，就一定可以确认 store 发生在 load 之前。(为了简单起见，我假设只有一个线程可以将 &a 写入 message。我们所说的 "thread 2 读取 thread 1 所写的值 "并不是指具体的那个数据进入了内存，它的真正意思是 thread 1 的 store 操作是可以影响到 thread 2 的所有操作中的最后一个），这样一来就形成了下面的顺序关系：

```sh
    a.x = 1;
       |
       v
    smp_store_release(&message, &a);  ----->  datum = smp_load_acquire(&message);
                                                  |
                                                  v
                                              datum->x
```

这样一来就一切可控了。由于可传递性的效果，每当 thread 2 读取 thread 1 写入的值时，thread 1 在 store-release 之前所做的一切操作在 load-acquire 操作之后对 thread 2 也全部都是可见的。需要注意的是，与 pthread_join()的情况不同，这里说的"synchronize with" 的说法并不意味着 thread 2 在 "等待" thread 1 完成写入操作。上面这幅顺序图中的情况其实只是 thread 2 恰好读到了 thread 1 写入值的情况。

在 Linux 内核中，上述代码的编写方式一般会有点不一样：

```sh
    thread 1                              thread 2
    --------------------------------      ------------------------
    a.x = 1;
    smp_wmb();
    WRITE_ONCE(message, &a);              datum = READ_ONCE(message);
                                          smp_rmb();
                                          if (datum != NULL)
                                            printk("%x\n", datum->x);
```

在这种写法下，释放和获取语义是由两个 memory barrier 操作 smp_wmb()和 smp_rmb() 所提供的。memory barrier 也有获取和释放的语义，但它们比简单的 load 和 store 操作更复杂一些。等我们今后介绍 seqlocks 时会再讲讲它们。

不管是 load-acquire/store-release 还是 smp_rmb()/smp_wmb()，都是非常常见的写法，大家需要确保理解。如下这些地方都用到了这些方式：

- 各种 ring buffer 方案。ring buffer 中每一项往往都指向其他的数据，一般来说还会有一些 head/tail 的信息都是指向 ring buffer 中某个位置的索引。给 ring buffer 填数据的一方（producer）会使用 store-release 操作，这样就能与 consumer（读取 ring buffer 中的数据的一方）确保同步。
- RCU。在 compiler 看来，我们熟悉的 rcu_dereference() 和 rcu_assign_pointer() API 跟 load-acquire 和 store-release 操作是一个效果。得益于除了 Alpha 以外的其他处理器中都具有的一些特性，rcu_dereference() 就可以被直接编译成普通的 load 操作，此时 rcu_assign_pointer() 仍然跟 rcu_dereference()操作是确保同步的，就好像它还是一个 load-acquire 操作一样。
- 将指针写入数组时。在下面的 KVM 代码中（当然是简写过的），如果 kvm_get_vcpu()可能看到 kvm->online_vcpus 增长过了，那么数组中与之关联的哪一项就一定是有效的。

```sh
    kvm_vm_ioctl_create_vcpu()                     kvm_get_vcpu()
    -----------------------------------------      -----------------------------------------------
    kvm->vcpus[kvm->online_vcpus] = vcpu;          if (idx < smp_load_acquire(&kvm->online_vcpus))
    smp_store_release(&kvm->online_vcpus,            return kvm->vcpus[idx];
                      kvm->online_vcpus + 1);      return NULL;
```

除了 load-acquire/store-release 操作机制外，message-passing 模式中还有一点值得注意：它是一种只有单个 producer（生产者）的算法。如果有多个 writer 的话，就必须通过其他手段（比如用 mutex）来进行互斥保护。lockless 算法并不是一个无法跟其他工具配合的东西，它们是并发编程工具(concurrent programming toolbox) 的一部分，当它们与其他传统工具结合时效果最好。
