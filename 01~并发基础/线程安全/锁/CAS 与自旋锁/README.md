# 乐观锁（Optimistic Locking）

相对悲观锁而言，乐观锁（Optimistic Locking）机制采取了更加宽松的加锁机制。相对悲观锁而言，乐观锁假设认为数据一般情况下不会造成冲突，所以在数据进行提交更新的时候，才会正式对数据的冲突与否进行检测，如果发现冲突了，则让返回用户错误的信息，让用户决定如何去做。上面提到的乐观锁的概念中其实已经阐述了他的具体实现细节：主要就是两个步骤：冲突检测和数据更新。其实现方式有一种比较典型的就是 Compare and Swap。

# CAS 与 ABA

CAS 是项乐观锁技术，当多个线程尝试使用 CAS 同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败，失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。CAS 操作包含三个操作数：内存位置(V)、预期原值(A)和新值(B)。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。无论哪种情况，它都会在 CAS 指令之前返回该位置的值。CAS 有效地说明了我认为位置 V 应该包含值 A；如果包含该值，则将 B 放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。这其实和乐观锁的冲突检查+数据更新的原理是一样的。

乐观锁也不是万能的，乐观并发控制相信事务之间的数据竞争(Data Race)的概率是比较小的，因此尽可能直接做下去，直到提交的时候才去锁定，所以不会产生任何锁和死锁。但如果直接简单这么做，还是有可能会遇到不可预期的结果，例如两个事务都读取了数据库的某一行，经过修改以后写回数据库，这时就遇到了问题。

- 乐观锁只能保证一个共享变量的原子操作。如上例子，自旋过程中只能保证 value 变量的原子性，这时如果多一个或几个变量，乐观锁将变得力不从心，但互斥锁能轻易解决，不管对象数量多少及对象颗粒度大小。
- 长时间自旋可能导致开销大。假如 CAS 长时间不成功而一直自旋，会给 CPU 带来很大的开销。
- ABA 问题。

CAS 的核心思想是通过比对内存值与预期值是否一样而判断内存值是否被改过，但这个判断逻辑不严谨，假如内存值原来是 A，后来被一条线程改为 B，最后又被改成了 A，则 CAS 认为此内存值并没有发生改变，但实际上是有被其他线程改过的，这种情况对依赖过程值的情景的运算结果影响很大。解决的思路是引入版本号，每次变量更新都把版本号加一。部分乐观锁的实现是通过版本号(version)的方式来解决 ABA 问题，乐观锁每次在执行数据的修改操作时，都会带上一个版本号，一旦版本号和数据的版本号一致就可以执行修改操作并对版本号执行 +1 操作，否则就执行失败。因为每次操作的版本号都会随之增加，所以不会出现 ABA 问题，因为版本号只会增加不会减少。

# 自旋锁

Linux 内核中最常见的锁，作用是在多核处理器间同步数据。这里的自旋是忙等待的意思。如果一个线程(这里指的是内核线程)已经持有了一个自旋锁，而另一条线程也想要获取该锁，它就不停地循环等待，或者叫做自旋等待直到锁可用。可以想象这种锁不能被某个线程长时间持有，这会导致其他线程一直自旋，消耗处理器。所以，自旋锁使用范围很窄，只允许短期内加锁。

自旋锁简单来说是一种低级的同步机制，表示了一个变量可能的两个状态：

- `acquired`;
- `released`.

每一个想要获取`自旋锁`的处理，必须为这个变量写入一个表示`自旋锁获取 (spinlock acquire)`状态的值，并且为这个变量写入`锁释放 (spinlock released)`状态。如果一个处理程序尝试执行受`自旋锁`保护的代码，那么代码将会被锁住，直到占有锁的处理程序释放掉。在本例中，所有相关的操作必须是 [原子的 (atomic)](https://en.wikipedia.org/wiki/Linearizability)，来阻止[竞态条件](https://en.wikipedia.org/wiki/Race_condition)状态。`自旋锁`在 Linux 内核中使用 `spinlock_t` 类型来表示。如果我们查看 Linux 内核代码，我们会看到，这个类型被[广泛地 (widely)](http://lxr.free-electrons.com/ident?i=spinlock_t) 使用。`spinlock_t`的定义如下：

```
typedef struct spinlock {
        union {
              struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
                struct {
                        u8 __padding[LOCK_PADSIZE];
                        struct lockdep_map dep_map;
                };
#endif
        };
} spinlock_t;
```

这段代码在 [include/linux/spinlock_types.h](https://github.com/torvalds/linux/master/include/linux/spinlock_types.h) 头文件中定义。可以看出，它的实现依赖于 `CONFIG_DEBUG_LOCK_ALLOC` 内核配置选项这个状态。

其实还有一种方式就是让等待线程睡眠直到锁可用，这样就可以消除忙等待。很明显后者优于前者的实现，但是却不适用于此，如果我们使用第二种方式，我们要做几步操作：把该等待线程换出、等到锁可用在换入，有两次上下文切换的代价。这个代价和短时间内自旋(实现起来也简单)相比，后者更能适应实际情况的需要。还有一点需要注意，试图获取一个已经持有自旋锁的线程再去获取这个自旋锁或导致死锁，但其他操作系统并非如此。

自旋锁与互斥锁有点类似，只是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是 否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。其作用是为了解决某项资源的互斥使用。因为自旋锁不会引起调用者睡眠，所以自旋锁的效率远 高于互斥锁。虽然它的效率比互斥锁高，但是它也有些不足之处：

- 自旋锁一直占用 CPU，他在未获得锁的情况下，一直运行－－自旋，所以占用着 CPU，如果不能在很短的时 间内获得锁，这无疑会使 CPU 效率降低。
- 在用自旋锁时有可能造成死锁，当递归调用时有可能造成死锁，调用有些其他函数也可能造成死锁，如 copy_to_user()、copy_from_user()、kmalloc()等。

自旋锁比较适用于锁使用者保持锁时间比较短的情况。正是由于自旋锁使用者一般保持锁时间非常短，因此选择自旋而不是睡眠是非常必要的，自旋锁的效率远高于互斥锁。信号量和读写信号量适合于保持时间较长的情况，它们会导致调用者睡眠，因此只能在进程上下文使用，而自旋锁适合于保持时间非常短的情况，它可以在任何上下文使用。如果被保护的共享资源只在进程上下文访问，使用信号量保护该共享资源非常合适，如果对共享资源的访问时间非常短，自旋锁也可以。但是如果被保护的共享资源需要在中断上下文访问(包括底半部即中断处理句柄和顶半部即软中断)，就必须使用自旋锁。自旋锁保持期间是抢占失效的，而信号量和读写信号量保持期间是可以被抢占的。自旋锁只有在内核可抢占或 SMP(多处理器)的情况下才真正需要，在单 CPU 且不可抢占的内核下，自旋锁的所有操作都是空操作。另外格外注意一点：自旋锁不能递归使用。

自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环查看是否该自旋锁的保持者已经释放了锁，"自旋"就是"在原地打转"。而信号量则引起调用者睡眠，它把进程从运行队列上拖出去，除非获得锁。这就是它们的"不类"。

互斥锁的起始原始开销要高于自旋锁，但是基本是一劳永逸，临界区持锁时间的大小并不会对互斥锁的开销造成影响，而自旋锁是死循环检测，加锁全程消耗 cpu，起始开销虽然低于互斥锁，但是随着持锁时间，加锁的开销是线性增长。

互斥锁用于临界区持锁时间比较长的操作，比如下面这些情况都可以考虑
1 临界区有 IO 操作
2 临界区代码复杂或者循环量大
3 临界区竞争非常激烈
4 单核处理器
至于自旋锁就主要用在临界区持锁时间非常短且 CPU 资源不紧张的情况下。

自旋锁是专为防止多处理器并发而引入的一种锁，它在内核中大量应用于中断处理等部分(对于单处理器来说，防止中断处理中的并发可简单采用关闭中断的方式，不需要自旋锁)。
自旋锁最多只能被一个内核任务持有，如果一个内核任务试图请求一个已被争用(已经被持有)的自旋锁，那么这个任务就会一直进行忙循环——旋转——等待锁重 新可用。要是锁未被争用，请求它的内核任务便能立刻得到它并且继续进行。自旋锁可以在任何时刻防止多于一个的内核任务同时进入临界区，因此这种锁可有效地 避免多处理器上并发运行的内核任务竞争共享资源。
事实上，自旋锁的初衷就是：在短期间内进行轻量级的锁定。一个被争用的自旋锁使得请求它的线程在等待锁重新可用的期间进行自旋(特别浪费处理器时间)，所以自旋锁不应该被持有时间过长。如果需要长时间锁定的话, 最好使用信号量。
自旋锁的基本形式如下：
spin_lock(&mr_lock);
//临界区
spin_unlock(&mr_lock);

因为自旋锁在同一时刻只能被最多一个内核任务持有，所以一个时刻只有一个线程允许存在于临界区中。这点很好地满足了对称多处理机器需要的锁定服务。在单处理器上，自旋锁仅仅当作一个设置内核抢占的开关。如果内核抢占也不存在，那么自旋锁会在编译时被完全剔除出内核。
简单的说，自旋锁在内核中主要用来防止多处理器中并发访问临界区，防止内核抢占造成的竞争。另外自旋锁不允许任务睡眠(持有自旋锁的任务睡眠会造成自死锁——因为睡眠有可能造成持有锁的内核任务被重新调度，而再次申请自己已持有的锁)，它能够在中断上下文中使用。
死锁：假设有一个或多个内核任务和一个或多个资源，每个内核都在等待其中的一个资源，但所有的资源都已经被占用了。这便会发生所有内核任务都在相互等待，但它们永远不会释放已经占有的资源，于是任何内核任务都无法获得所需要的资源，无法继续运行，这便意味着死锁发生了。自死琐是说自己占有了某个资源，然后 自己又申请自己已占有的资源，显然不可能再获得该资源，因此就自缚手脚了。

自旋锁与互斥锁有点类似，只是自旋锁不会引起调用者睡眠，如果自旋锁已经被别的执行单元保持，调用者就一直循环在那里看是 否该自旋锁的保持者已经释放了锁，"自旋"一词就是因此而得名。其作用是为了解决某项资源的互斥使用。因为自旋锁不会引起调用者睡眠，所以自旋锁的效率远 高于互斥锁。虽然它的效率比互斥锁高，但是它也有些不足之处：
1、自旋锁一直占用 CPU，他在未获得锁的情况下，一直运行－－自旋，所以占用着 CPU，如果不能在很短的时 间内获得锁，这无疑会使 CPU 效率降低。
2、在用自旋锁时有可能造成死锁，当递归调用时有可能造成死锁，调用有些其他函数也可能造成死锁，如 copy_to_user()、copy_from_user()、kmalloc()等。
因此我们要慎重使用自旋锁，自旋锁只有在内核可抢占式或 SMP 的情况下才真正需要，在单 CPU 且不可抢占式的内核下，自旋锁的操作为空操作。自旋锁适用于锁使用者保持锁时间比较短的情况下。
自旋锁的用法如下：
首先定义：spinlock_t x;
然后初始化：spin_lock_init(spinlock_t \*x); //自旋锁在真正使用前必须先初始化
在 2.6.11 内核中将定义和初始化合并为一个宏：DEFINE_SPINLOCK(x)

获得自旋锁：spin_lock(x); //只有在获得锁的情况下才返回，否则一直“自旋”
spin_trylock(x); //如立即获得锁则返回真，否则立即返回假
释放锁：spin_unlock(x);

结合以上有以下代码段：

    spinlock_t lock;        //定义一个自旋锁
    spin_lock_init(&lock);
    spin_lock(&lock);
    .......        //临界区
    spin_unlock(&lock);   //释放锁

    还有一些其他用法：

spin_is_locked(x)
//　　该宏用于判断自旋锁 x 是否已经被某执行单元保持(即被锁)，如果是，返回真，否则返回假。
spin_unlock_wait(x)
//　　该宏用于等待自旋锁 x 变得没有被任何执行单元保持，如果没有任何执行单元保持该自旋锁，该宏立即返回，否
//将循环 在那里，直到该自旋锁被保持者释放。

spin_lock_irqsave(lock, flags)
//　　该宏获得自旋锁的同时把标志寄存器的值保存到变量 flags 中并失效本地中//断。相当于：spin_lock()+local_irq_save()
spin_unlock_irqrestore(lock, flags)
//　　该宏释放自旋锁 lock 的同时，也恢复标志寄存器的值为变量 flags 保存的//值。它与 spin_lock_irqsave 配对使用。
//相当于：spin_unlock()+local_irq_restore()

spin_lock_irq(lock)
　 //该宏类似于 spin_lock_irqsave，只是该宏不保存标志寄存器的值。相当 //于：spin_lock()+local_irq_disable()
spin_unlock_irq(lock)
//该宏释放自旋锁 lock 的同时，也使能本地中断。它与 spin_lock_irq 配对应用。相当于: spin_unlock()+local_irq+enable()

spin_lock_bh(lock)
//　　该宏在得到自旋锁的同时失效本地软中断。相当于: //spin_lock()+local_bh_disable()
spin_unlock_bh(lock)
//该宏释放自旋锁 lock 的同时，也使能本地的软中断。它与 spin_lock_bh 配对//使用。相当于：spin_unlock()+local_bh_enable()

spin_trylock_irqsave(lock, flags)
//该宏如果获得自旋锁 lock，它也将保存标志寄存器的值到变量 flags 中，并且失//效本地中断，如果没有获得锁，它什么也不做。因此如果能够立即 获得锁，它等//同于 spin_lock_irqsave，如果不能获得锁，它等同于 spin_trylock。如果该宏//获得自旋锁 lock，那需要 使用 spin_unlock_irqrestore 来释放。

spin_trylock_irq(lock)
//该宏类似于 spin_trylock_irqsave，只是该宏不保存标志寄存器。如果该宏获得自旋锁 lock，需要使用 spin_unlock_irq 来释放。
spin_trylock_bh(lock)
//　　该宏如果获得了自旋锁，它也将失效本地软中断。如果得不到锁，它什么//也不做。因此，如果得到了锁，它等同于 spin_lock_bh，如果得 不到锁，它等同//于 spin_trylock。如果该宏得到了自旋锁，需要使用 spin_unlock_bh 来释放。
spin_can_lock(lock)
//　　该宏用于判断自旋锁 lock 是否能够被锁，它实际是 spin_is_locked 取反。//如果 lock 没有被锁，它返回真，否则，返回 假。该宏在 2.6.11 中第一次被定义，在//先前的内核中并没有该宏。

So now you know how to implement reasonable spinlocks. This may come handy for example when you're writing for a platform with glibc that doesn't have pthread_spin_lock, like Mac OS X. Here's the shim for that case:

```cpp
int pthread_spin_init(pthread_spinlock_t *lock, int pshared) {
    __asm__ __volatile__ ("" ::: "memory");
    *lock = 0;
    return 0;
}

int pthread_spin_destroy(pthread_spinlock_t *lock) {
    return 0;
}

int pthread_spin_lock(pthread_spinlock_t *lock) {
    while (1) {
        int i;
        for (i=0; i < 10000; i++) {
            if (__sync_bool_compare_and_swap(lock, 0, 1)) {
                return 0;
            }
        }
        sched_yield();
    }
}

int pthread_spin_trylock(pthread_spinlock_t *lock) {
    if (__sync_bool_compare_and_swap(lock, 0, 1)) {
        return 0;
    }
    return EBUSY;
}

int pthread_spin_unlock(pthread_spinlock_t *lock) {
    __asm__ __volatile__ ("" ::: "memory");
    *lock = 0;
    return 0;
}
```

自旋锁与互斥量类似，但它不是通过休眠使进程阻塞，而是在获取锁之前一直处于忙等（自旋）阻塞状态。自旋锁可以用于以下情况：锁被持有的时间短，而且线程并不希望在重新调度上花费太多的成本。

# 乐观锁 CAS

在 JDK1.5 中新增 `java.util.concurrent` (J.U.C)就是建立在 CAS 之上的。相对于对于 synchronized 这种阻塞算法，CAS 是非阻塞算法的一种常见实现。所以 J.U.C 在性能上有了很大的提升。我们以 `java.util.concurrent` 中的 `AtomicInteger` 为例，看一下在不使用锁的情况下是如何保证线程安全的。主要理解 `getAndIncrement` 方法，该方法的作用相当于 `++i` 操作。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
  private volatile int value;

  public final int get() {
    return value;
  }

  public final int getAndIncrement() {
    for (;;) {
      int current = get();
      int next = current + 1;
      if (compareAndSet(current, next)) return current;
    }
  }

  public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
  }
}
```

在没有锁的机制下需要字段 value 要借助 volatile 原语，保证线程间的数据是可见的。这样在获取变量的值的时候才能直接读取。然后来看看 `++i` 是怎么做到的。`getAndIncrement` 采用了 CAS 操作，每次从内存中读取数据然后将此数据和 `+1` 后的结果进行 CAS 操作，如果成功就返回结果，否则重试直到成功为止。而 `compareAndSet` 利用 JNI 来完成 CPU 指令的操作。

# 自旋锁

自旋锁是采用让当前线程不停地的在循环体内执行实现的，当循环的条件被其他线程改变时才能进入临界区。

```java
public class SpinLock {
  private AtomicReference<Thread> sign = new AtomicReference<>();

  public void lock() {
    Thread current = Thread.currentThread();
    while (!sign.compareAndSet(null, current)) {}
  }

  public void unlock() {
    Thread current = Thread.currentThread();
    sign.compareAndSet(current, null);
  }
}
```

使用了 CAS 原子操作，lock 函数将 owner 设置为当前线程，并且预测原来的值为空。unlock 函数将 owner 设置为 null，并且预测值为当前线程。当有第二个线程调用 lock 操作时由于 owner 值不为空，导致循环一直被执行，直至第一个线程调用 unlock 函数将 owner 设置为 null，第二个线程才能进入临界区。由于自旋锁只是将当前线程不停地执行循环体，不进行线程状态的改变，所以响应速度更快。但当线程数不停增加时，性能下降明显，因为每个线程都需要执行，占用 CPU 时间。如果线程竞争不激烈，并且保持锁的时间段。适合使用自旋锁。

# Links

- [深入理解乐观锁与悲观锁](http://www.hollischuang.com/archives/934)
