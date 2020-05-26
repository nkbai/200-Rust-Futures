##   引言

这本书的目的是用一个例子驱动的方法来解释Rust中的Futures，探索为什么他们被设计成这样，以及他们如何工作。 我们还将介绍在编程中处理并发性时的一些替代方案。

理解这本书中描述的细节， 不需要使用Rust中的 futures或async/await。 这是为那些好奇的人准备的，他们想知道这一切是如何运作的。


### 这本书涵盖的内容

这本书将试图解释你可能想知道的一切，包括不同类型的执行器(executor)和运行时(runtime)的主题。 我们将在本书中实现一个非常简单的运行时，介绍一些概念，但这已经足够开始了。


[Stjepan Glavina](https://github.com/stjepang) 发表了一系列关于异步运行时和执行器的优秀文章，如果谣言属实，那么在不久的将来他会发表更多的文章。

你应该做的是先读这本书，然后继续阅读 [stejpang 的文章](https://stjepang.github.io/)，了解更多关于运行时以及它们是如何工作的，特别是:
1. [构建自的block_on](https://stjepang.github.io/2020/01/25/build-your-own-block-on.html)
2. [构建自己的executor](https://stjepang.github.io/2020/01/31/build-your-own-executor.html)

 我将自己限制在一个200行的主示例(因此才有了这个标题)中，以限制范围并介绍一个可以轻松进一步研究的示例。

 然而，有很多东西需要消化，这并不像我所说的那么简单，但是我们会一步一步来，所以喝杯茶，放松一下。

> 这本书是在开放的，并欢迎贡献。 你可以在[这里找到这本书](https://github.com/cfsamson/books-futures-explained)。 最后的例子，你可以[克隆或复制](https://github.com/cfsamson/examples-futures)可以在这里找到。 任何建议或改进可以归档为一个公关或在问题追踪的书。
> 一如既往，我们欢迎各种各样的反馈。


### 阅读练习和进一步阅读
 
在[最后一章](#结论和练习))中，我冒昧地提出了一些小练习，如果你想进一步探索的话。

这本书也是我在 Rust 中写的关于并发编程的第四本书。 如果你喜欢它，你可能也想看看其他的:

- [Green Threads Explained in 200 lines of rust](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)
- [The Node Experiment - Exploring Async Basics with Rust 节点实验——用Rust探索异步基础](https://cfsamson.github.io/book-exploring-async-basics/)
- [Epoll, Kqueue and IOCP Explained with Rust 用 Rust 解释 Epoll，Kqueue 和 IOCP](https://cfsamsonbooks.gitbook.io/epoll-kqueue-iocp-explained/)

### 感谢

我想借此机会感谢 mio，tokio，async std，Futures，libc，crossbeam 背后的人们，他们支撑着这个异步生态系统，却很少在我眼中得到足够的赞扬。

特别感谢 jonhoo，他对我这本书的初稿给予了一些有价值的反馈。 他还没有读完最终的成品，但是我们应该向他表示感谢。

