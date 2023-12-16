# BlockingQueue 实现的生产者与消费者

在传统的生产者消费者模型中，通常是采用 BlockingQueue 实现。其中生产者线程负责提交需求，消费者线程负责处理任务，二者之间通过共享内存缓冲区进行通信。由于内存缓冲区的存在，允许生产者和消费者之间速度的差异，确保系统正常运行。下例展示一个简单的生产者消费者模型，生产者从文件中读取数据，将数据内容写入到阻塞队列中，消费者从队列的另一边获取数据，进行计算并将结果输出。其中 Main 负责创建两类线程并初始化队列。

```java
public class Main {
    public static void main(String[] args) {
        // 初始化阻塞队列
        BlockingQueue<String> blockingQueue = new ArrayBlockingQueue<>(1000);
        // 创建生产者线程
        Thread producer = new Thread(new Producer(blockingQueue, "temp.dat"));
        producer.start();
        // 创建消费者线程
        Thread consumer = new Thread(new Consumer(blockingQueue));
        consumer.start();
    }
}

// 生产者
public class Producer implements Runnable {
    private BlockingQueue<String> blockingQueue;
    private String fileName;
    private static final String FINIDHED = "EOF";

    public Producer(BlockingQueue<String> blockingQueue, String fileName)  {
        this.blockingQueue = blockingQueue;
        this.fileName = fileName;
    }

    @Override
    public void run() {
        try {
            BufferedReader reader = new BufferedReader(new FileReader(new File(fileName)));
            String line;
            while ((line = reader.readLine()) != null) {
                blockingQueue.put(line);
            }
            // 结束标志
            blockingQueue.put(FINIDHED);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

// 消费者
public class Consumer implements Runnable {
    private BlockingQueue<String> blockingQueue;
    private static final String FINIDHED = "EOF";

    public Consumer(BlockingQueue<String> blockingQueue) {
        this.blockingQueue = blockingQueue;
    }

    @Override
    public void run() {
        String line;
        String[] arrStr;
        int ret;
        try {
            while (!(line = blockingQueue.take()).equals(FINIDHED)) {
                // 消费
                arrStr = line.split("\t");
                if (arrStr.length != 2) {
                    continue;
                }
                ret = Integer.parseInt(arrStr[0]) + Integer.parseInt(arrStr[1]);
                System.out.println(ret);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

上述使用了 ArrayBlockingQueue，通过查看其实现，完全是使用锁和阻塞等待实现线程同步。在高并发场景下，性能不是很优越。

# Disruptor 实现

Disruptor 要求数组大小设置为 2 的 N 次方。这样可以通过 Seq & (QueueSize - 1) 直接获取，其效率要比取模快得多。这是因为（Queue - 1）的二进制为全 1 等形式。例如，QueueSize 大小为 8，Seq 为 10，则只需要计算二进制 1010 & 0111 = 2，可直接得到 index=2 位置的元素。在 RingBuffer 中，生产者向数组中写入数据，生产者写入数据时，使用 CAS 操作。消费者从中读取数据时，为防止多个消费者同时处理一个数据，也使用 CAS 操作进行数据保护。这种固定大小的 RingBuffer 还有一个好处是，可以内存复用。不会有新空间需要分配或者旧的空间回收，当数组填充满后，再写入数据会将数据覆盖。

由于我们只需要将文件中的数据行读出，然后进行计算。因此，定义 FileData.class 来保存文件行。

```java
public class FileData {
    private String line;

    public String getLine() {
        return line;
    }

    public void setLine(String line) {
        this.line = line;
    }
}
```

然后用于产生 FileData 的工厂类，会在 Disruptor 系统初始化时，构造所有的缓冲区中的对象实例。

```java
public class DisruptorFactory implements EventFactory<FileData> {

    public FileData newInstance() {
        return new FileData();
    }
}
```

接下来消费者的作用是读取数据并进行处理。数据的读取已经由 Disruptor 封装，onEvent()方法为 Disruptor 框架的回调方法。只需要进行简单的数据处理即可。

```java
public class DisruptorConsumer implements WorkHandler<FileData> {
    private static final String FINIDHED = "EOF";

    @Override
    public void onEvent(FileData event) throws Exception {
       String line = event.getLine();
        if (line.equals(FINIDHED)) {
            return;
        }
        // 消费
        String[] arrStr = line.split("\t");
        if (arrStr.length != 2) {
            return;
        }
        int ret = Integer.parseInt(arrStr[0]) + Integer.parseInt(arrStr[1]);
        System.out.println(ret);
    }
}

```

生产者需要一个 Ringbuffer 的引用。其中 pushData()方法是将生产的数据写入到 RingBuffer 中。具体的过程是，首先通过 next()方法得到下一个可用的序列号；取得下一个可用的 FileData，并设置该对象的值；最后，进行数据发布，这个 FileData 对象会传递给消费者。

```java
public class DisruptorProducer {
    private static final String FINIDHED = "EOF";
    private final RingBuffer<FileData> ringBuffer;

    public DisruptorProducer(RingBuffer<FileData> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    public void pushData(String line) {
        long seq = ringBuffer.next();
        try {
            FileData event = ringBuffer.get(seq);   // 获取可用位置
            event.setLine(line);                    // 填充可用位置
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            ringBuffer.publish(seq);        // 通知消费者
        }
    }

    public void read(String fileName) {
        try {
            BufferedReader reader = new BufferedReader(new FileReader(new File(fileName)));
            String line;
            while ((line = reader.readLine()) != null) {
                // 生产数据
                pushData(line);
            }
            // 结束标志
            pushData(FINIDHED);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

最后需要一个 DisruptorMain 将上述的数据、生产者和消费者进行整合。

```java
public class DisruptorMain {
    public static void main(String[] args) {
        DisruptorFactory factory = new DisruptorFactory();          // 工厂
        ExecutorService executor = Executors.newCachedThreadPool(); // 线程池
        int BUFFER_SIZE = 16;   // 必须为2的幂指数

        // 初始化Disruptor
        Disruptor<FileData> disruptor = new Disruptor<>(factory,
                BUFFER_SIZE,
                executor,
                ProducerType.MULTI,         // Create a RingBuffer supporting multiple event publishers to the one RingBuffer
                new BlockingWaitStrategy()  // 默认阻塞策略
                );

        // 启动消费者
        disruptor.handleEventsWithWorkerPool(new DisruptorConsumer(),
                new DisruptorConsumer()
        );
        disruptor.start();

        // 启动生产者
        RingBuffer<FileData> ringBuffer = disruptor.getRingBuffer();
        DisruptorProducer producer = new DisruptorProducer(ringBuffer);
        producer.read("temp.dat");

        // 关闭
        disruptor.shutdown();
        executor.shutdown();
    }
}
```

## Disruptor 策略

Disruptor 生产者和消费者之间是通过什么策略进行同步呢？Disruptor 提供了如下几种策略：

- BlockingWaitStrategy：默认等待策略。和 BlockingQueue 的实现很类似，通过使用锁和条件（Condition）进行线程同步和唤醒。此策略对于线程切换来说，最节约 CPU 资源，但在高并发场景下性能有限。
- SleepingWaitStrategy：CPU 友好型策略。会在循环中不断等待数据。首先进行自旋等待，若不成功，则使用 Thread.yield()让出 CPU，并使用 LockSupport.parkNanos(1)进行线程睡眠。所以，此策略数据处理数据可能会有较高的延迟，适合用于对延迟不敏感的场景。优点是对生产者线程影响小，典型应用场景是异步日志。
- YieldingWaitStrategy：低延时策略。消费者线程会不断循环监控 RingBuffer 的变化，在循环内部使用 Thread.yield()让出 CPU 给其他线程。
- BusySpinWaitStrategy：死循环策略。消费者线程会尽最大可能监控缓冲区的变化，会占用所有 CPU 资源。
