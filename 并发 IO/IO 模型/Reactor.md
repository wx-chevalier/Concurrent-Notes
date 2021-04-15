# 线程模型

在[《Linux 线程模型》](https://ng-tech.icu/Linux-Series/#/)中我们讨论过 Linux 中线程的实现以及用户态线程与内核态线程中等概念，在这里我们讨论的线程模型，则着眼于如何高效的利用多个物理核，进行工作任务的调度，使得系统能够有更高有效的吞吐，更加低的延迟。而不是被某个等待 IO 的线程阻塞或者把时间花在大量的比如系统层面的上下文切换等工作。

网络 IO 中最简单的就是连接独占模型：也就是一个连接进来请求后，独占一个线程（进程）进行处理；无论其中中间在做什么事情，比如调用第三方的服务，等待过程中也是独占着整个线程。传统的如 Tomcat Servlet 就是基于这样的原理。而我们现在实际应用中常用的是 Reactor 模型，单线程 Reactor 模型譬如使用单个线程处理所有连接上的请求，使用 epoll-wait 方式，实现事件多路复用机制；典型比如 Redis，适用于简单比如小数据的内存数据的获取，每一个回调逻辑都比较简单。还有多线程 Reactor 模型，即多个线程/进程 Accept 同一个连接上的请求。

> 实践案例可以参阅《[Database-Series/Redis](https://github.com/wx-chevalier/Database-Series?q=)》等相关章节。

# Reactor 处理逻辑

在 Reactor 模型中，一般具备以下组件：

- Synchronous Event Demultiplexer：同步事件分离器，用于监听各种事件，调用方调用监听方法的时候会被阻塞，直到有事件发生，才会返回。对于 Linux 来说，同步事件分离器指的就是 IO 多路复用模型，比如 epoll，poll 等， 对于 Java NIO 来说， 同步事件分离器对应的组件就是 selector，对应的阻塞方法就是 select。
- Handler：本质上是文件描述符，是一个抽象的概念，可以简单的理解为一个一个事件，该事件可以来自于外部，比如客户端连接事件，客户端的写事件等等，也可以是内部的事件，比如操作系统产生的定时器事件等等。
- Event Handler：事件处理器，本质上是回调方法，当有事件发生后，框架会根据 Handler 调用对应的回调方法，在大多数情况下，是虚函数，需要用户自己实现接口，实现具体的方法。
- Concrete Event Handler： 具体的事件处理器，是 Event Handler 的具体实现。
- Initiation Dispatcher：初始分发器，实际上就是 Reactor 角色，提供了一系列方法，对 Event Handler 进行注册和移除；还会调用 Synchronous Event Demultiplexer 监听各种事件；当有事件发生后，还要调用对应的 Event Handler。

![Reactor 核心组件](https://pic.imgdb.cn/item/6077b6f98322e6675c23f11e.jpg)

Reactor 模型的基本的处理逻辑为：

- 我们注册 Concrete Event Handler 到 Initiation Dispatcher 中。
- Initiation Dispatcher 调用每个 Event Handler 的 get_handle 接口获取其绑定的 Handle。
- Initiation Dispatcher 调用 handle_events 开始事件处理循环。在这里，Initiation Dispatcher 会将步骤 2 获取的所有 Handle 都收集起来，使用 Synchronous Event Demultiplexer 来等待这些 Handle 的事件发生。
- 当某个(或某几个)Handle 的事件发生时，Synchronous Event Demultiplexer 通知 Initiation Dispatcher。
- Initiation Dispatcher 根据发生事件的 Handle 找出所对应的 Handler。
- Initiation Dispatcher 调用 Handler 的 handle_event 方法处理事件。

抽象来说，Reactor 有 4 个核心的操作：

- add：添加 socket 监听到 reactor，可以是 listen socket 也可以使客户端 socket，也可以是管道、eventfd、信号等
- set：修改事件监听，可以设置监听的类型，如可读、可写。可读很好理解，对于 listen socket 就是有新客户端连接到来了需要 accept。对于客户端连接就是收到数据，需要 recv。可写事件比较难理解一些。一个 SOCKET 是有缓存区的，如果要向客户端连接发送 2M 的数据，一次性是发不出去的，操作系统默认 TCP 缓存区只有 256K。一次性只能发 256K，缓存区满了之后 send 就会返回 EAGAIN 错误。这时候就要监听可写事件，在纯异步的编程中，必须去监听可写才能保证 send 操作是完全非阻塞的。
- del：从 reactor 中移除，不再监听事件
- callback：就是事件发生后对应的处理逻辑，一般在 add/set 时制定。C 语言用函数指针实现，JS 可以用匿名函数，PHP 可以用匿名函数、对象方法数组、字符串函数名。

Reactor 只是一个事件发生器，实际对 socket 句柄的操作，如 connect/accept、send/recv、close 是在 callback 中完成的。具体编码可参考下面的 Swoole 伪代码：

![Swoole 伪代码](https://pic.imgdb.cn/item/6077b80d8322e6675c25caae.jpg)

Reactor 模型还可以与多进程、多线程结合起来用，既实现异步非阻塞 IO，又利用到多核。目前流行的异步服务器程序都是这样的方式：如

- Nginx：多进程 Reactor
- Nginx+Lua：多进程 Reactor+协程
- Golang：单线程 Reactor+多线程协程
- Swoole：多线程 Reactor+多进程 Worker

协程从底层技术角度看实际上还是异步 IO Reactor 模型，应用层自行实现了任务调度，借助 Reactor 切换各个当前执行的用户态线程，但用户代码中完全感知不到 Reactor 的存在。

# Reactor 模型

在 BIO 模型中，主要依赖多开线程的方式来提供服务端的吞吐。但是多开线程势必带来的问题就是系统层面的开销比较大（contex-switch、cache-bounding 等等），对于高性能场景典型就不太适用。如前所介绍的 Reactor 模型，我们天然地会引入多线程来解决这个问题。

Doug Lea 把 Reactor 模式分为三种类型，分别是：Basic version, Multi-threaded version 和 Multi-reactors version。Reactor 模型在 Linux 系统中的具体实现即是 select/poll/epoll/kqueue，像 Redis 中即是采用了 Reactor 模型实现了单进程单线程高并发。

## 单线程模型

所有的 IO 操作都在同一个 NIO 线程上面完成，NIO 线程需要处理客户端的 TCP 连接，同时读取客户端 Socket 的请求或者应答消息以及向客户端发送请求或者应答消息。如下图：

![基础版本](https://s3.ax1x.com/2021/03/01/6P2vzn.png)

所有的 I/O 操作都在同一个 NIO 线程上面完成。可以看到有多个客户端连接到 Reactor，Reactor 内部有一个 dispatch（分发器）。有连接请求后，Reactor 会通过 dispatch 把请求交给 Acceptor 进行处理，有 IO 读写事件之后，又会通过 dispatch 交给具体的 Handler 进行处理。此时一个 Reactor 既然负责处理连接请求，又要负责处理读写请求，一般来说处理连接请求是很快的，但是处理具体的读写请求就要涉及到业务逻辑处理了，相对慢太多了。Reactor 正在处理读写请求的时候，其他请求只能等着，只有等处理完了，才可以处理下一个请求。

该模型的处理时序图如下:

![Reactor 单线程模型时序图](https://i.postimg.cc/zfNqBwz2/65cdba67cfcee3302b88d114e2fd5baf.png)

单线程模型只适用并发量比较小的应用场景。当一个 NIO 线程同时处理上万个连接时，处理速度会变慢，会导致大量的客户端连接超时，超时之后如果进行重发，更加会加重了 NIO 线程的负载，最终会有大量的消息积压和处理超时，NIO 线程会成为系统的瓶颈。对于一些小容量应用场景，可以使用单线程模型，但是对于高负载、大并发的应用却不合适。实际当中基本不会采用单线程模型。

### 代码案例

```java
public class Main {

    public static void main(String[] args) {
        Reactor reactor = new Reactor(9090);
        reactor.run();
    }
}

public class Reactor implements Runnable {

    ServerSocketChannel serverSocketChannel;
    Selector selector;

    // 在构造方法中，注册了连接事件，并且在 selectionKey 对象附加了一个 Acceptor 对象，这是用来处理连接请求的类。
    public Reactor(int port) {
        try {
            serverSocketChannel = ServerSocketChannel.open();
            selector = Selector.open();
            serverSocketChannel.socket().bind(new InetSocketAddress(port));
            serverSocketChannel.configureBlocking(false);
            // 注册了连接事件
            SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            // 并且在 selectionKey 对象附加了一个 Acceptor 对象，这是用来处理连接请求的类
            // 发生了连接事件后，Reactor 类的 dispatcher 方法拿到了 Acceptor 附加对象，调用了 Acceptor 的 run 方法
            selectionKey.attach(new Acceptor(selector, serverSocketChannel));

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Reactor类实现了Runnable接口，并且实现了run方法，在run方法中，监听各种事件，有了事件后，调用dispatcher方法
    @Override
    public void run() {
        while (true) {
            try {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    dispatcher(selectionKey);
                    iterator.remove();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    // 在 dispatcher 方法中，拿到了 selectionKey 附加的对象，随后调用 run 方法，注意此时是调用 run 方法，并没有开启线程，只是一个普通的调用而已。
    private void dispatcher(SelectionKey selectionKey) {
        Runnable runnable = (Runnable) selectionKey.attachment();
        runnable.run();
    }
}

public class Acceptor implements Runnable {
    private Selector selector;

    private ServerSocketChannel serverSocketChannel;

    public Acceptor(Selector selector, ServerSocketChannel serverSocketChannel) {
        this.selector = selector;
        this.serverSocketChannel = serverSocketChannel;
    }

    // 在 Acceptor 的 run 方法中又注册了读事件，然后在 selectionKey 附加了一个 WorkHandler 对象；
    // Acceptor 的 run 方法执行完毕后，就会继续回到 Reactor 类中的 run 方法，负责监听事件。
    @Override
    public void run() {
        try {
            SocketChannel socketChannel = serverSocketChannel.accept();
            System.out.println("有客户端连接上来了," + socketChannel.getRemoteAddress());
            socketChannel.configureBlocking(false);
            SelectionKey selectionKey = socketChannel.register(selector, SelectionKey.OP_READ);
            // 当客户端写事件发生后，Reactor 又会调用 dispatcher 方法，此时拿到的附加对象是WorkHandler，所以又跑到了 WorkHandler 中的run方法。
            selectionKey.attach(new WorkerHandlerThreadPool(socketChannel));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

// WorkHandler就是真正负责处理客户端写事件的了
public class WorkHandler implements Runnable {
    private SocketChannel socketChannel;

    public WorkHandler(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    @Override
    public void run() {
        try {
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            socketChannel.read(byteBuffer);
            String message = new String(byteBuffer.array(), StandardCharsets.UTF_8);
            System.out.println(socketChannel.getRemoteAddress() + "发来的消息是:" + message);
            //System.out.println("sleep 10s");
            //Thread.sleep(10000);

            socketChannel.write(ByteBuffer.wrap("你的消息我收到了".getBytes(StandardCharsets.UTF_8)));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

对应的测试用客户端代码如下：

```java
public class Client {
    public static void main(String[] args) {
        try {
            Socket socket = new Socket();
            socket.connect(new InetSocketAddress("localhost", 9090));
            new Thread(() -> {
                while (true) {
                    try {
                        InputStream inputStream = socket.getInputStream();
                        byte[] bytes = new byte[1024];
                        inputStream.read(bytes);
                        System.out.println(new String(bytes, StandardCharsets.UTF_8));
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            }).start();

            while (true) {
                Scanner scanner = new Scanner(System.in);
                while (scanner.hasNextLine()) {
                    String s = scanner.nextLine();
                    socket.getOutputStream().write(s.getBytes());
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

/*
    有客户端连接上来了,/127.0.0.1:64571
    有客户端连接上来了,/127.0.0.1:64577
    /127.0.0.1:64571发来的消息是:123
    /127.0.0.1:64577发来的消息是:456
    /127.0.0.1:64571发来的消息是:我是第一个人
    /127.0.0.1:64577发来的消息是:我是第二个人
*/
```

## 多线程模型

多线程模型与单线程模型最大的区别是有专门的一组 NIO 线程处理 IO 操作，一般使用线程池的方式实现。一个 NIO 线程同时处理多条连接，一个连接只能属于 1 个 NIO 线程，这样能够防止并发操作问题。

![Reactor 模式的多线程版本](https://s3.ax1x.com/2021/03/01/6PRCZT.png)

多线程模型中有一个专门的线程负责监听和处理所有的客户端连接，网络 IO 操作由一个 NIO 线程池负责，它包含一个任务队列和 N 个可用的线程，由这些 NIO 线程负责消息的读取、解码、编码和发送。1 个 NIO 线程可以同时处理 N 条链路，但是 1 个链路只对应 1 个 NIO 线程，防止发生并发操作问题。在绝大多数场景下，Reactor 多线程模型都可以满足性能需求。但是在更高并发连接的场景（如百万客户端并发连接），它的性能似乎不能满足需求，于是便出现了下面的多 Reactor（主从 Reactor）模型。

### 代码案例

```java
public class Acceptor implements Runnable {
    private Selector selector;

    private ServerSocketChannel serverSocketChannel;

    public Acceptor(Selector selector, ServerSocketChannel serverSocketChannel) {
        this.selector = selector;
        this.serverSocketChannel = serverSocketChannel;
    }

    @Override
    public void run() {
        try {
            SocketChannel socketChannel = serverSocketChannel.accept();
            System.out.println("有客户端连接上来了," + socketChannel.getRemoteAddress());
            socketChannel.configureBlocking(false);
            SelectionKey selectionKey = socketChannel.register(selector, SelectionKey.OP_READ);
            // 修改此处就可以切换成多线程模型了
            selectionKey.attach(new WorkerHandlerThreadPool(socketChannel));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class WorkerHandlerThreadPool implements Runnable {

    static ExecutorService pool = Executors.newFixedThreadPool(2);

    private SocketChannel socketChannel;

    public WorkerHandlerThreadPool(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    @Override
    public void run() {
        try {
            System.out.println("WorkerHandlerThreadPool thread :" + Thread.currentThread().getName());
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            socketChannel.read(buffer);
            pool.execute(new Process(socketChannel, buffer));
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class Process implements Runnable {

    private SocketChannel socketChannel;

    private ByteBuffer byteBuffer;

    public Process(SocketChannel socketChannel, ByteBuffer byteBuffer) {
        this.byteBuffer = byteBuffer;
        this.socketChannel = socketChannel;
    }

    @Override
    public void run() {
        try {
            System.out.println("sleep 10s");
            Thread.sleep(10000);
            System.out.println("process thread:" + Thread.currentThread().getName());
            String message = new String(byteBuffer.array(), StandardCharsets.UTF_8);
            System.out.println(socketChannel.getRemoteAddress() + "发来的消息是:" + message);
            socketChannel.write(ByteBuffer.wrap("你的消息我收到了".getBytes(StandardCharsets.UTF_8)));
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}

/**
    有客户端连接上来了,/127.0.0.1:64797
    有客户端连接上来了,/127.0.0.1:64800
    WorkerHandlerThreadPool thread :main
    sleep 10s
    process thread:pool-1-thread-1
    /127.0.0.1:64797发来的消息是:I am one
    WorkerHandlerThreadPool thread :main
    sleep 10s
    WorkerHandlerThreadPool thread :main
    sleep 10s
    process thread:pool-1-thread-2
    /127.0.0.1:64800发来的消息是:I am two
    process thread:pool-1-thread-1
    /127.0.0.1:64797发来的消息是:hah
*/
```

单 Reactor 多线程模型看起来是很不错了，但是还是有缺点：一个 Reactor 还是既然负责连接请求，又要负责读写请求，连接请求是很快的，而且一个客户端一般只要连接一次就可以了，但是会发生很多次写请求，如果可以有多个 Reactor，其中一个 Reactor 负责处理连接事件，多个 Reactor 负责处理客户端的写事件就好了，这样更符合单一职责，所以主从 Reactor 模型诞生了。

## 主从多线程模型

服务端用于接收客户端连接的不是 1 个单独的 NIO 线程了，而是采用独立的 NIO 线程池。Acceptor 接收 TCP 连接请求处理完成之后，将创建新的 SocketChannel 注册到处理连接的 IO 线程池中的某个 IO 线程上，有它去处理 IO 读写以及编解码的工作。Acceptor 只用于客户端登录、握手以及认证，一旦连接成功之后，将链路注册到线程池的 IO 线程上。

![主从 Reactor 模型](https://s3.ax1x.com/2021/03/01/6PRAJJ.png)

这种模型是将 Reactor 分成两部分，mainReactor 负责监听 server socket、accept 新连接，并将建立的 socket 分派给 subReactor；subReactor 负责多路分离已连接的 socket，读写网络数据；而对业务处理的功能，交给 worker 线程池来完成。

### 代码案例

```java
public class Reactor implements Runnable {

    ServerSocketChannel serverSocketChannel;
    Selector selector;

    public Reactor(int port) {
        try {
            serverSocketChannel = ServerSocketChannel.open();
            selector = Selector.open();
            serverSocketChannel.socket().bind(new InetSocketAddress(port));
            serverSocketChannel.configureBlocking(false);
            SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
            selectionKey.attach(new Acceptor(serverSocketChannel));

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while (true) {
            try {
                selector.select();
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    dispatcher(selectionKey);
                    iterator.remove();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void dispatcher(SelectionKey selectionKey) {
        Runnable runnable = (Runnable) selectionKey.attachment();
        runnable.run();
    }
}

public class Acceptor implements Runnable {
    private ServerSocketChannel serverSocketChannel;
    private final int core = 8;
    private int index;
    private SubReactor[] subReactors = new SubReactor[core];
    private Thread[] threads = new Thread[core];
    private final Selector[] selectors = new Selector[core];

    public Acceptor(ServerSocketChannel serverSocketChannel) {
        this.serverSocketChannel = serverSocketChannel;
        for (int i = 0; i < core; i++) {
            try {
                selectors[i] = Selector.open();
            } catch (IOException e) {
                e.printStackTrace();
            }
            subReactors[i] = new SubReactor(selectors[i]);
            threads[i] = new Thread(subReactors[i]);
            threads[i].start();
        }
    }

    @Override
    public void run() {
        try {
            System.out.println("acceptor thread:" + Thread.currentThread().getName());

            SocketChannel socketChannel = serverSocketChannel.accept();
            System.out.println("有客户端连接上来了," + socketChannel.getRemoteAddress());
            socketChannel.configureBlocking(false);
            selectors[index].wakeup();
            SelectionKey selectionKey = socketChannel.register(selectors[index], SelectionKey.OP_READ);
            selectionKey.attach(new WorkerHandler(socketChannel));
            if (++index == threads.length) {
                index = 0;
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

public class SubReactor implements Runnable {

    private Selector selector;

    public SubReactor(Selector selector) {
        this.selector = selector;
    }

    @Override
    public void run() {
        while (true) {
            try {
                selector.select();
                System.out.println("selector:" + selector.toString() + "thread:" + Thread.currentThread().getName());
                Set<SelectionKey> selectionKeys = selector.selectedKeys();
                Iterator<SelectionKey> iterator = selectionKeys.iterator();
                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    dispacher(selectionKey);
                    iterator.remove();
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void dispacher(SelectionKey selectionKey) {
        Runnable runnable = (Runnable) selectionKey.attachment();
        runnable.run();
    }
}

public class WorkerHandler implements Runnable {
    private SocketChannel socketChannel;

    public WorkerHandler(SocketChannel socketChannel) {
        this.socketChannel = socketChannel;
    }

    @Override
    public void run() {
        try {
            System.out.println("workHandler thread:" + Thread.currentThread().getName());
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            socketChannel.read(buffer);
            String message = new String(buffer.array(), StandardCharsets.UTF_8);
            System.out.println(socketChannel.getRemoteAddress() + "发来的消息：" + message);
            socketChannel.write(ByteBuffer.wrap("你的消息我收到了".getBytes(StandardCharsets.UTF_8)));

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

/**
acceptor thread:main
有客户端连接上来了,/127.0.0.1:65194
selector:sun.nio.ch.KQueueSelectorImpl@5a506132thread:Thread-0
selector:sun.nio.ch.KQueueSelectorImpl@5a506132thread:Thread-0
workHandler thread:Thread-0
/127.0.0.1:65194发来的消息：123

acceptor thread:main
有客户端连接上来了,/127.0.0.1:65202
selector:sun.nio.ch.KQueueSelectorImpl@59887d72thread:Thread-1
selector:sun.nio.ch.KQueueSelectorImpl@59887d72thread:Thread-1
workHandler thread:Thread-1
/127.0.0.1:65202发来的消息：444
**/
```

可以很清楚的看到，从始至终，acceptor 都只有一个 main 线程，而负责处理客户端写请求的是不同的线程，而且还是不同的 reactor、selector。

# Links

- https://blog.csdn.net/u013074465/article/details/46276967
