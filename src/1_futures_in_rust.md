##  Rust中的Futures

### 概述

1. Rust中并发性的高级介绍
2. 了解 Rust 在使用异步代码时能提供什么，不能提供什么
3. 了解为什么我们需要 Rust 的运行时库
4. 理解“leaf-future”和“non-leaf-future”的区别
5. 了解如何处理 CPU 密集型任务


### Futures

什么是`Future`?
`Future`是一些将在未来完成的操作。
Rust中的异步实现基于轮询,每个异步任务分成三个阶段:
1. 轮询阶段(The Poll phase). 一个`Future`被轮询后,会开始执行,直到被阻塞. 我们经常把轮询一个Future这部分称之为执行器(executor)
2. 等待阶段.  事件源(通常称为reactor)注册等待一个事件发生，并确保当该事件准备好时唤醒相应的`Future`
3. 唤醒阶段.  事件发生,相应的`Future`被唤醒。 现在轮到执行器(executor),就是第一步中的那个执行器，调度`Future`再次被轮询，并向前走一步，直到它完成或达到一个阻塞点，不能再向前走, 如此往复,直到最终完成.

当我们谈论`Future`的时候，我发现在早期区分`non-leaf-future`和`leaf-future`是很有用的，因为实际上它们彼此很不一样。


### Leaf futures

由运行时创建`leaf futures`，它就像套接字一样,代表着一种资源.
```rust
// stream is a **leaf-future**
let mut stream = tokio::net::TcpStream::connect("127.0.0.1:3000");
```
对这些资源的操作，比如套接字上的 Read 操作，将是非阻塞的，并返回一个我们称之为`leaf-future`的Future.之所以称之为`leaf-future`,是因为这是我们实际上正在等待的Future.

除非你正在编写一个运行时，否则你不太可能自己实现一个`leaf-future`，但是我们将在本书中详细介绍它们是如何构造的。

您也不太可能将 `leaf-future` 传递给运行时，然后单独运行它直到完成，这一点您可以通过阅读下一段来理解。

### Non-leaf-futures

Non-leaf-futures指的是那些我们用`async`关键字创建的Future.

异步程序的大部分是Non-leaf-futures，这是一种可暂停的计算。 这是一个重要的区别，因为这些`Future`代表一组操作。 通常，这样的任务由`await` 一系列`leaf-future`组成.

```rust
// Non-leaf-future
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap();// <- yield
    println!("connected!");
    let result = stream.write(b"hello world\n").await; // <- yield
    println!("message sent!");
    ...
};
```

这些任务的关键是，它们能够将控制权交给运行时的调度程序，然后在稍后停止的地方继续执行。
与`leaf-future`相比，这些Future本身并不代表I/O资源。 当我们对这些Future进行轮询时, 有可能会运行一段时间或者因为等待相关资源而让度给调度器,然后等待相关资源ready的时候唤醒自己.



### 运行时(Runtimes)

像 c # ，JavaScript，Java，GO 和许多其他语言都有一个处理并发的运行时。 所以如果你来自这些语言中的一种，这对你来说可能会有点奇怪。

Rust 与这些语言的不同之处在于 Rust 没有处理并发性的运行时，因此您需要使用一个为您提供此功能的库。

很多复杂性归因于 Futures 实际上是来源于运行时的复杂性，创建一个有效的运行时是困难的。
学习如何正确使用一个也需要相当多的努力，但是你会看到这些类型的运行时之间有几个相似之处，所以学习一个可以使学习下一个更容易。

Rust 和其他语言的区别在于，在选择运行时时，您必须进行主动选择。大多数情况下，在其他语言中，你只会使用提供给你的那一种。

异步运行时可以分为两部分:
1. 执行器(The Executor)
2. reactor (The Reactor)

当 Rusts Futures 被设计出来的时候，有一个愿望，那就是将通知`Future`它可以做更多工作的工作与`Future`实际做工作分开。

你可以认为前者是reactor的工作，后者是执行器的工作。 运行时的这两个部分使用 `Waker`进行交互。

写这篇文章的时候，未来最受欢迎的两个运行时是:

1. [async-std](https://github.com/async-rs/async-std)
2. [Tokio](https://github.com/tokio-rs/tokio)

### Rust 的标准库做了什么

1. 一个公共接口，`Future trait`
2. 一个符合人体工程学的方法创建任务, 可以通过async和await关键字进行暂停和恢复`Future`
3. `Waker`接口, 可以唤醒暂停的`Future`

这就是Rust标准库所做的。 正如你所看到的，不包括异步I/O的定义,这些异步任务是如何被创建的,如何运行的。

### I/O密集型 VS CPU密集型任务

正如你们现在所知道的，你们通常所写的是`Non-leaf-futures`。 让我们以 pseudo-rust 为例来看一下这个异步块:
```rust
let non_leaf = async {
    let mut stream = TcpStream::connect("127.0.0.1:3000").await.unwrap(); // <-- yield
    
    // request a large dataset
    let result = stream.write(get_dataset_request).await.unwrap(); // <-- yield
    
    // wait for the dataset
    let mut response = vec![];
    stream.read(&mut response).await.unwrap(); // <-- yield

    // do some CPU-intensive analysis on the dataset
    let report = analyzer::analyze_data(response).unwrap();
    
    // send the results back
    stream.write(report).await.unwrap(); // <-- yield
};

```

现在，正如您将看到的，当我们介绍 Futures 的工作原理时，两个`yield`之间的代码与我们的执行器在同一个线程上运行。

这意味着当我们分析器处理数据集时，执行器忙于计算而不是处理新的请求。

幸运的是，有几种方法可以解决这个问题，这并不困难，但是你必须意识到:
1. 我们可以创建一个新的`leaf future`，它将我们的任务发送到另一个线程，并在任务完成时解析。 我们可以像等待其他Future一样等待这个`leaf-future`。
2. 运行时可以有某种类型的管理程序来监视不同的任务占用多少时间，并将执行器本身移动到不同的线程，这样即使我们的分析程序任务阻塞了原始的执行程序线程，它也可以继续运行。
3. 您可以自己创建一个与运行时兼容的`reactor`，以您认为合适的任何方式进行分析，并返回一个可以等待的未来。

现在，#1是通常的处理方式，但是一些执行器也实现了#2。 2的问题是，如果你切换运行时，你需要确保它也支持这种监督，否则你最终会阻塞执行者。

方式#3更多的是理论上的重要性，通常您会很乐意将任务发送到多数运行时提供的线程池。

大多数执行器都可以使用诸如 spawn blocking 之类的方法来完成#1。

这些方法将任务发送到运行时创建的线程池，在该线程池中，您可以执行 cpu 密集型任务，也可以执行运行时不支持的“阻塞”任务。

现在，有了这些知识，你已经在一个很好的方式来理解`Future`，但我们不会停止，有很多细节需要讨论。

休息一下或喝杯咖啡，准备好我们进入下一章的深度探索。

### 奖励部分


如果你发现并发和异步编程的概念一般来说令人困惑，我知道你是从哪里来的，我已经写了一些资源，试图给出一个高层次的概述，这将使之后更容易学习 Rusts Futures:

- [Async Basics - The difference between concurrency and parallelism 异步基础-并发和并行之间的区别](https://cfsamson.github.io/book-exploring-async-basics/1_concurrent_vs_parallel.html)
- [Async Basics - Async history 异步基础-异步历史](https://cfsamson.github.io/book-exploring-async-basics/2_async_history.html)
- [Async Basics - Strategies for handling I/O 异步基础-处理 i / o 的策略](https://cfsamson.github.io/book-exploring-async-basics/5_strategies_for_handling_io.html)
- [Async Basics - Epoll, Kqueue and IOCP 异步基础-Epoll，Kqueue 和 IOCP](https://cfsamson.github.io/book-exploring-async-basics/6_epoll_kqueue_iocp.html)


通过研究`Future`来学习这些概念会让它变得比实际需要难得多，所以如果你有点不确定的话，继续读这些章节。

你回来的时候我就在这儿。

如果你觉得你已经掌握了基本知识，那么让我们开始行动吧！
