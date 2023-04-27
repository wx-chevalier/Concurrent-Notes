# 可见性与缓存一致性

所谓的可见性，即是一个线程对共享变量的修改，另外一个线程能够立刻看到。单核时代，所有的线程都是直接操作单个 CPU 的数据，某个线程对缓存的写对另外一个线程来说一定是可见的；譬如下图中，如果线程 B 在线程 A 更新了变量值之后进行访问，那么获得的肯定是变量 V 的最新值。多核时代，每颗 CPU 都有自己的缓存，共享变量存储在主内存。运行在某个 CPU 中的线程将共享变量读取到自己的 CPU 缓存。在 CPU 缓存中，修改了共享对象的值，由于 CPU 并未将缓存中的数据刷回主内存，导致对共享变量的修改对于在另一个 CPU 中运行的线程而言是不可见的。这样每个线程都会拥有一份属于自己的共享变量的拷贝，分别存于各自对应的 CPU 缓存中。

![CPU、内存与变量](https://assets.ng-tech.icu/item/20230426174201.png)

# Links

- https://surfingcomplexity.blog/2022/11/25/cache-invalidation-really-is-one-of-the-hardest-things-in-computer-science/
