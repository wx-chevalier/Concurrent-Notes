[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![license: CC BY-NC-SA 4.0](https://img.shields.io/badge/license-CC%20BY--NC--SA%204.0-lightgrey.svg)][license-url]

<!-- PROJECT LOGO -->
<br />
<p align="center">
  <a href="https://github.com/wx-chevalier/Concurrent-Series">
    <img src="https://s2.ax1x.com/2019/09/04/nEBum6.png" alt="Logo">
  </a>

  <h3 align="center">Concurrent Series</h3>

  <p align="center">
    📚深入浅出并发编程实践：并发基础、并发控制、并发模型、并发 IO
    <br />
    <a href="https://github.com/wx-chevalier/Concurrent-Series"><strong>在线阅读 >> </strong></a>
    <br />
    <br />
    <a href="https://github.com/wx-chevalier/Concurrent-Series">速览手册</a>
    ·
    <a href="https://github.com/wx-chevalier/Concurrent-Series/issues">Report Bug</a>
    ·
    <a href="https://github.com/wx-chevalier/Concurrent-Series/issues">参考资料</a>
  </p>
</p>

<!-- ABOUT THE PROJECT -->

# Introduction | 前言

随着处理器技术的发展，单核时代以提升处理器频率来提高运行效率的方式遇到了瓶颈，目前各种主流的 CPU 频率基本被锁定在了 3GHZ 附近。单核 CPU 的发展的停滞，给多核 CPU 的发展带来了机遇。相应地，编程语言也开始逐步向并行化的方向发展。为了让代码运行得更快，单纯依靠更快的硬件已无法满足要求，并行和分布式计算是现代应用程序的主要内容；我们需要利用多个核心或多台机器来加速应用程序或大规模运行它们，并发编程日益成为编程中不可忽略的重要组成部分。

简单定义来看，如果执行单元的逻辑控制流在时间上重叠，那它们就是并发（Concurrent）的；而从分布式一致性模型的定义来看，如果两个操作都没有在彼此之前发生，那么这两个操作是并发的。换句话说，如果两个事件是因果相关的（一个发生在另一个事件之前），则它们之间是有序的，但如果它们是并发的，则它们之间的顺序是无法比较的。这意味着因果关系定义了一个偏序，而不是一个全序：一些操作相互之间是有顺序的，但有些则是无法比较的。由这些定义可扩展到非常广泛的概念，其向下依赖于操作系统、存储等，与分布式系统、微服务等，而又会具体落地于 Java 并发编程、Go 并发编程、JavaScript 异步编程等领域。云计算承诺在所有维度上（内存、计算、存储等）实现无限的可扩展性，并发编程及其相关理论也是我们构建大规模分布式应用的基础。

![并发编程](https://s2.ax1x.com/2019/09/02/nCL9Ej.png)

本篇主要讨论并发编程理论相关的内容，其精排目录导航版请参考 [https://ng-tech.icu/Concurrent-Series](https://ng-tech.icu/Concurrent-Series)。本篇专注于通用的并发理论知识，各语言级别实现可以参考 [Java 并发编程](http://ng-tech.icu/Java-Series/)、[Go 并发编程](http://ng-tech.icu/Go-Series/)、[JavaScript 异步编程](http://ng-tech.icu/JavaScript-Series/)等。

# Nav | 关联导航

## 应用场景

从计算机系统本身而言，并发被看做是操作系统内核用来运行多个应用程序的机制。很久很久以前是没有并发这个概念的，因为那个时候操作系统并不支持多任务。现在的操作系统今非昔比，支持抢占式任务、多线程、分页、TCP/IP 等现代操作系统特性，能满足用户各种各样的需求同时响应用户的不同操作，靠的是多任务系统的支持。在 Unix 时代一个进程(执行中的程序)只有一个线程，现代操作系统允许一个进程有多条线程。每个线程有独立栈空间、寄存器、程序计数器(存放下一条执行指令)。操作系统调度程序调度的是线程(在本文中提到线程的地方都指的是操作系统级的线程)，并非进程，也就是说线程才是调度的基本单位。

在单处理器系统中，一个处于运行状态的 IO 消耗型程序，例如一个网络读取阻塞或用户输入阻塞导致处理器出现空闲，在视处理器为昂贵资源的情形下，这是巨大的浪费。然后，在对称处理器中(SMP)，因为只有一个线程，那它只能在其中一个处理器上运行，也就是说剩余的处理器被白白浪费。如果使用多线程，不仅能充分利用多处理器，还能通过线程并发解决上述 IO 消耗程序中处理器空闲问题。并发编程能够解决的典型问题如下所示：

```c
while (true){
     request = next_http_request()
     request_work(request)
}
```

当 request_work 发起 IO 之后 CPU 是完全空闲下来的，而可怜的新请求(next_http_request)必须等待 IO 完成之后才可以获取 CPU 的控制权。所以 CPU 的利用率非常低，并发要解决的问题就是提高 CPU 的利用率。明白这一点我们也就清楚了，并发只对非 CPU 密集型程序管用，如果 CPU 利用率非常高，更多的并发只会让情况更加糟糕。并发虽好，也不能滥用啊，典型的并发编程的适用场景包括:

- 在多处理器上进行并行计算：在只有一个 CPU 的单处理器上, 井发流是交替的。在任何时间点上, 都只有一个流在 CPU 上实际执行。然而, 那些有多个 CPU 的机器, 称为多处理器, 可以真正地同时执行多个流。被分成并发流的并发应用 在这样的机器上能够运行得快很多，这对大规模数据库和科学应用尤为重要。

- 访问慢速 IO 设备：当一个应用正在等待来自慢速 IO 设备 (例如磁盘) 的数据到达时，内核会运行其他进程使 CPU 保持繁忙。每个应用都可以以类似的方式，通过交替执行 IO，请求和其他有用的工作来使用并发性。

- 与人交互：和计算机交互的人要求计算机同时执行多个任务的能力。例如, 他们在打印个文档时, 可能想要调整一个窗口的大伽 现代视窗系统利用并发性来提供这种能九 每次用户请求某种操作 忧匕如说通过单击鼠标) 时, 一个独立的并发逻辑流被创建来执行这个操作。

- 通过推迟工作以减少执行时延：有时, 应用程序能够通过推迟其他操作井同时执行它们,利用并发性来降低某些操作的延遮 比如, ˉ 一个动态存储分配器可以通过椎迟与酬个运行在较低优先级上的并发 “合井” 流的合并 (coalescing), 使用空闲时的 CPU 周期, 来降低单个 free 操作的延迟。

- 服务多个网络客户端：一个慢速的客户端可能会导致服务器拒绝为所有其他客户端服务。对于闻个真正的服务器来说 可能期望它每秒为成百上千的客户端提供服务, 某些个慢速客户端寻致拒绝为其他客户端服务, 这是不能接受峋 一个更好的方法是创建一个并发服务糕 它为每个客户端创建各自独立的逻辑流 这就允许服务器同时为多个客户端服务, 并且这也避免了慢速客户端独占服务器。

## 困难与挑战

在并发程序中，遇到资源竞争时，为了保证线程安全性，通常会引入某种同步手段，保证任意时刻只有一个线程访问资源。但是这种情况下，其它线程会因为等待资源而被挂起，延长总体执行时间，可能会引起线程活跃性问题。我们引入并发程序的目的是为了提高程序的执行性能，活跃性问题让我们与目标背道而驰。

值得一提的是，并发也绝非银弹，在多核的前提下，性能和线程是紧密联系在一起的。线程间的跳转对高频 IO 操作的性能有决定性作用: 一次跳转意味着至少 3-20 微秒的延时，由于每个核心的 L1/L2 Cache 独，随之而来是大量的 cache miss，一些变量的读取、写入延时会从纳秒级上升几百倍至微秒级: 等待 CPU 把对应的缓存行同步过来。有时这带来了一个出乎意料的结果，当每次的处理都很简短时，一个多线程程序未必比一个单线程程序更快。因为前者可能在每次付出了大的切换代价后只做了一点点“正事”，而后者在不停地做“正事”。不过单线程也是有代价的，它工作良好的前提是“正事”都很快，否则一旦某次变慢就使后续的所有“正事”都被延迟了。在一些处理时间普遍较短的程序中，使用（多个不相交的）单线程能最大程度地”做正事“，由于每个请求的处理时间确定，延时表现也很稳定，各种 HTTP 服务器正是这样。但我们的检索服务要做的事情可就复杂多了，有大量的后端服务需要访问，广泛存在的长尾请求使每次处理的时间无法确定，排序策略也越来越复杂。如果还是使用（多个不相交的）单线程的话，一次难以预计的性能抖动，或是一个大请求可能导致后续一堆请求被延迟。

从编程模型上来看，并发多线程往往也容易引入更多的问题，特别是对基于共享内存与锁的并发模型中，其主要面对如下的问题：

- “抢占“式的线程切换：你无法确定两个线程访问数据的顺序，一切都很随机

- “同步“不可组装：同步的代码组装起来也不同步，必须加个更大的同步块

# About | 关于

<!-- CONTRIBUTING -->

## Contributing

Contributions are what make the open source community such an amazing place to be learn, inspire, and create. Any contributions you make are **greatly appreciated**.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

<!-- ACKNOWLEDGEMENTS -->

## Acknowledgements

- [Awesome-Lists](https://github.com/wx-chevalier/Awesome-Lists): 📚 Guide to Galaxy, curated, worthy and up-to-date links/reading list for ITCS-Coding/Algorithm/SoftwareArchitecture/AI. 💫 ITCS-编程/算法/软件架构/人工智能等领域的文章/书籍/资料/项目链接精选。

- [Awesome-CS-Books](https://github.com/wx-chevalier/Awesome-CS-Books): :books: Awesome CS Books/Series(.pdf by git lfs) Warehouse for Geeks, ProgrammingLanguage, SoftwareEngineering, Web, AI, ServerSideApplication, Infrastructure, FE etc. :dizzy: 优秀计算机科学与技术领域相关的书籍归档。

## Copyright & More | 延伸阅读

笔者所有文章遵循[知识共享 署名 - 非商业性使用 - 禁止演绎 4.0 国际许可协议](https://creativecommons.org/licenses/by-nc-nd/4.0/deed.zh)，欢迎转载，尊重版权。您还可以前往 [NGTE Books](https://ng-tech.icu/books/) 主页浏览包含知识体系、编程语言、软件工程、模式与架构、Web 与大前端、服务端开发实践与工程架构、分布式基础架构、人工智能与深度学习、产品运营与创业等多类目的书籍列表：

[![NGTE Books](https://s2.ax1x.com/2020/01/18/19uXtI.png)](https://ng-tech.icu/books/)

<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->

[contributors-shield]: https://img.shields.io/github/contributors/wx-chevalier/Concurrent-Series.svg?style=flat-square
[contributors-url]: https://github.com/wx-chevalier/Concurrent-Series/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/wx-chevalier/Concurrent-Series.svg?style=flat-square
[forks-url]: https://github.com/wx-chevalier/Concurrent-Series/network/members
[stars-shield]: https://img.shields.io/github/stars/wx-chevalier/Concurrent-Series.svg?style=flat-square
[stars-url]: https://github.com/wx-chevalier/Concurrent-Series/stargazers
[issues-shield]: https://img.shields.io/github/issues/wx-chevalier/Concurrent-Series.svg?style=flat-square
[issues-url]: https://github.com/wx-chevalier/Concurrent-Series/issues
[license-shield]: https://img.shields.io/github/license/wx-chevalier/Concurrent-Series.svg?style=flat-square
[license-url]: https://github.com/wx-chevalier/Concurrent-Series/blob/master/LICENSE.txt
