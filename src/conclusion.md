##  结论和练习

我们的实现采取了一些明显的捷径，可以使用一些改进。 实际上，深入研究代码并自己尝试一些事情是一个很好的学习方式。 如果你想探索更多，这里有一些很好的练习

### 编码park

使用 Thread: : park 和 Thread: : unpark 的大问题是用户可以从自己的代码访问这些相同的方法。 尝试使用另一种方法来挂起线程并根据我们的命令再次唤醒它。 一些提示:

- [std::sync::CondVar](https://doc.rust-lang.org/stable/std/sync/struct.Condvar.html)
- [crossbeam::sync::Parker](https://github.com/crossbeam-rs/crossbeam)


### 编码传递reactor

避免包装整个Reactor in a mutex and pass it around 在互斥对象中传递

首先，保护整个reactor并传递它是过分的。 我们只对同步它包含的部分信息感兴趣。 尝试将其重构出来，只同步访问真正需要的内容。

我建议您看看[async_std驱动程序](https://github.com/async-rs/async-std/blob/master/src/net/driver/mod.rs)是如何实现的，以及[tokio调度程序](https://github.com/tokio-rs/tokio/blob/master/tokio/src/runtime/basic_scheduler.rs)是如何实现的，以获得一些灵感。

- 你想使用Arc来传递这些信息的引用?
- 你是否想创建一个全局的reactor,这样他可以随时随地被访问?


### 创建一个更好的执行器

现在我们只支持一个一个Future运行. 大多数运行时都有一个spawn函数,让你能够启动一个future,然后await它. 这样你就可以同时运行多个Future.

正如我在本书开头所建议的那样，访问[stjepan 的博客系列](https://stjepang.github.io/2020/01/31/build-your-own-executor.html)，了解如何实现你自己的执行者，是我将要开始的地方，并从那里开始。


### 进一步阅读

- [The official Asyc book](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)

- [ The async_std book](https://book.async.rs/)

- [Aron Turon: Designing futures for Rust](https://aturon.github.io/blog/2016/09/07/futures-design/)

- [Steve Klabnik's presentation: Rust's journey to Async/Await](https://www.infoq.com/presentations/rust-2019/)
  
- [The Tokio Blog](https://tokio.rs/blog/2019-10-scheduler/)

- [Stjepan's blog with a series where he implements an Executor](https://stjepang.github.io/)

- [Jon Gjengset's video on The Why, What and How of Pinning in Rust](https://youtu.be/DkMwYxfSYNQ)   有中英文字幕的[B站链接](https://www.bilibili.com/video/BV1tK411L7s5/)

- [Withoutboats blog series about async/await](https://boats.gitlab.io/blog/post/2018-01-25-async-i-self-referential-structs/)

