# IO 模型

按照《Unix 网络编程》的划分，IO 模型可以分为：阻塞 IO、非阻塞 IO、IO 复用、信号驱动 IO 和异步 IO，按照 POSIX 标准来划分只分为两类：同步 IO 和异步 IO。

首先一个 IO 操作(read/write 系统调用)其实分成了两个步骤：1)发起 IO 请求和 2)实际的 IO 读写(内核态与用户态的数据拷贝)阻塞 IO 和非阻塞 IO 的区别在于第一步，发起 IO 请求的进程是否会被阻塞，如果阻塞直到 IO 操作完成才返回那么就是传统的阻塞 IO，如果不阻塞，那么就是非阻塞 IO。同步 IO 和异步 IO 的区别就在于第二步，实际的 IO 读写(内核态与用户态的数据拷贝)是否需要进程参与，如果需要进程参与则是同步 IO，如果不需要进程参与就是异步 IO。如果实际的 IO 读写需要请求进程参与，那么就是同步 IO。因此阻塞 IO、非阻塞 IO、IO 复用、信号驱动 IO 都是同步 IO，在编程上，这种非阻塞 IO 一般都采用 IO 状态事件+回调方法的方式来处理 IO 操作。如果是同步 IO，则状态事件为读写就绪。此时的数据仍在内核态中，但是已经准备就绪，可以进行 IO 读写操作。如果是异步 IO，则状态事件为读写完成。此时的数据已经存在于应用进程的地址空间（用户态）中。

在 IO 类型划分中，最重要的就是阻塞/非阻塞、同步/异步这两组概念。阻塞模式的 IO 会造成应用程序等待，直到 IO 完成。同时操作系统也支持将 IO 操作设置为非阻塞模式，这时应用程序的调用将可能在没有拿到真正数据时就立即返回了，为此应用程序需要多次调用才能确认 IO 操作完全完成。而同步与异步，更多地是在应用层面的概念，如果做阻塞 IO 调用，应用程序等待调用的完成的过程就是一种同步状况。相反，IO 为非阻塞模式时，应用程序则是异步的。不过值得一提的是，在 Linux 的 IO 模型中，非阻塞与异步是不同的 IO 模型，这里的异步强调的是异步通知触发。

从应用的调度看，IO 还可以根据以下维度进行划分：

- 大/小块 IO：这个数值指的是控制器指令中给出的连续读出扇区数目的多少。如果数目较多，如 64，128 等，我们可以认为是大块 IO；反之，如果很小，比如 4，8，我们就会认为是小块 IO，实际上，在大块和小块 IO 之间，没有明确的界限。

- 连续/随机 IO：连续 IO 指的是本次 IO 给出的初始扇区地址和上一次 IO 的结束扇区地址是完全连续或者相隔不多的。反之，如果相差很大，则算作一次随机 IO。连续 IO 比随机 IO 效率高的原因是：在做连续 IO 的时候，磁头几乎不用换道，或者换道的时间很短；而对于随机 IO，如果这个 IO 很多的话，会导致磁头不停地换道，造成效率的极大降低。

- 顺序/并发 IO：从概念上讲，并发 IO 就是指向一块磁盘发出一条 IO 指令后，不必等待它回应，接着向另外一块磁盘发 IO 指令。对于具有条带性的 RAID（LUN），对其进行的 IO 操作是并发的，例如：raid 0+1(1+0),raid5 等。反之则为顺序 IO。