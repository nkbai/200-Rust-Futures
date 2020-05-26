##  实现Futures--主要例子

我们将用一个伪reactor和一个简单的执行器创建我们自己的`Futures`，它允许你在浏览器中编辑和运行代码

我将向您介绍这个示例，但是如果您想更深入的研究它，您可以[克隆存储库](https://github.com/cfsamson/examples-futures)并自己处理代码，或者直接从下一章复制代码。

readme文件中解释了几个分支，其中有两个分支与本章相关。 主分支是我们在这里经过的例子，`basic_example_commented`分支是这个具有大量注释的例子

> 如果您希望跟随我们的步骤，可以通过创建一个新的文件夹初始化一个新的 cargo 项目，并在其中运行 cargo init。所有的一切都在main.rs文件中.

### 实现我们自己的Futures

让我们先从引入依赖开始:

```rust
use std::{
    future::Future, pin::Pin, sync::{mpsc::{channel, Sender}, Arc, Mutex},
    task::{Context, Poll, RawWaker, RawWakerVTable, Waker},
    thread::{self, JoinHandle}, time::{Duration, Instant}
};
```

### 执行器

执行器的责任是获取一个或多个`Future`然后运行他们到完成。

执行器拿到`Future`后的第一件事就是轮询它.

轮询后可以发现以下三种情况:
1. `Future`返回Ready,然后就可以调度其他任何后续操作.
2. 这个`Future`从未被轮询过,所以传入一个`Waker`,然后将它挂起
3. 这个`Future`已经被轮询过,但是返回`Pending`

 

Rust通过`Waker`为Reactor和执行器提供了通信方式. reactor存储这个`Waker`,然后在`Future`等待的事件完成的时候调用`Waker: : wake ()`,这样`Future`就会被再次轮询.

我们的执行器会是这个样子:

```rust
// Our executor takes any object which implements the `Future` trait
fn block_on<F: Future>(mut future: F) -> F::Output {

    // the first thing we do is to construct a `Waker` which we'll pass on to
    // the `reactor` so it can wake us up when an event is ready. 
    let mywaker = Arc::new(MyWaker{ thread: thread::current() }); 
    let waker = waker_into_waker(Arc::into_raw(mywaker));

    // The context struct is just a wrapper for a `Waker` object. Maybe in the
    // future this will do more, but right now it's just a wrapper.
    let mut cx = Context::from_waker(&waker);

    // So, since we run this on one thread and run one future to completion
    // we can pin the `Future` to the stack. This is unsafe, but saves an
    // allocation. We could `Box::pin` it too if we wanted. This is however
    // safe since we shadow `future` so it can't be accessed again and will
    // not move until it's dropped.
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // We poll in a loop, but it's not a busy loop. It will only run when
    // an event occurs, or a thread has a "spurious wakeup" (an unexpected wakeup
    // that can happen for no good reason).
    let val = loop {
        
        match Future::poll(pinned, &mut cx) {

            // when the Future is ready we're finished
            Poll::Ready(val) => break val,

            // If we get a `pending` future we just go to sleep...
            Poll::Pending => thread::park(),
        };
    };
    val
}
```


在本章的所有例子中，我都选择了对代码进行广泛的注释。 我发现沿着这条路走会更容易一些，所以我不会在这里重复自己的话，只关注一些可能需要进一步解释的重要方面。

现在你已经阅读了这么多关于生成器和 Pin 的内容，这应该很容易理解。 `Future`是一个状态机，每一个`await`点也是一个`yield`点。 我们可以跨越`await`借用，我们遇到的问题与跨`yield`借用时完全一样。


> `Context`只是 `Waker` 的包装器, 至少在我写这本书的时候，它仅仅是这样。 在未来，`Context`对象可能不仅仅是包装一个`Waker`(译者注,原文是Future,应该有误)，因此这种额外的抽象可以提供一些灵活性。

正如在关于生成器的章节中解释的那样，我们使用Pin来保证允许`Future`有自引用。


### 实现Future
`Future`有一个定义良好的接口，这意味着他们可以用于整个生态系统。

我们可以将这些`Future`连接起来，这样一旦`leaf-future`准备好了，我们就可以执行一系列操作，直到任务完成或者我们到达另一个`leaf-future`，我们将等待并将控制权交给调度程序。


我们`Future`的实现是这样的:

```rust
// This is the definition of our `Waker`. We use a regular thread-handle here.
// It works but it's not a good solution. It's easy to fix though, I'll explain
// after this code snippet.
#[derive(Clone)]
struct MyWaker {
    thread: thread::Thread,
}

// This is the definition of our `Future`. It keeps all the information we
// need. This one holds a reference to our `reactor`, that's just to make
// this example as easy as possible. It doesn't need to hold a reference to
// the whole reactor, but it needs to be able to register itself with the
// reactor.
#[derive(Clone)]
pub struct Task {
    id: usize,
    reactor: Arc<Mutex<Box<Reactor>>>,
    data: u64,
}

// These are function definitions we'll use for our waker. Remember the
// "Trait Objects" chapter earlier.
fn mywaker_wake(s: &MyWaker) {
    let waker_ptr: *const MyWaker = s;
    let waker_arc = unsafe {Arc::from_raw(waker_ptr)};
    waker_arc.thread.unpark();
}

// Since we use an `Arc` cloning is just increasing the refcount on the smart
// pointer.
fn mywaker_clone(s: &MyWaker) -> RawWaker {
    let arc = unsafe { Arc::from_raw(s) };
    std::mem::forget(arc.clone()); // increase ref count
    RawWaker::new(Arc::into_raw(arc) as *const (), &VTABLE)
}

// This is actually a "helper funtcion" to create a `Waker` vtable. In contrast
// to when we created a `Trait Object` from scratch we don't need to concern
// ourselves with the actual layout of the `vtable` and only provide a fixed
// set of functions
const VTABLE: RawWakerVTable = unsafe {
    RawWakerVTable::new(
        |s| mywaker_clone(&*(s as *const MyWaker)),     // clone
        |s| mywaker_wake(&*(s as *const MyWaker)),      // wake
        |s| mywaker_wake(*(s as *const &MyWaker)),      // wake by ref
        |s| drop(Arc::from_raw(s as *const MyWaker)),   // decrease refcount
    )
};

// Instead of implementing this on the `MyWaker` object in `impl Mywaker...` we
// just use this pattern instead since it saves us some lines of code.
fn waker_into_waker(s: *const MyWaker) -> Waker {
    let raw_waker = RawWaker::new(s as *const (), &VTABLE);
    unsafe { Waker::from_raw(raw_waker) }
}

impl Task {
    fn new(reactor: Arc<Mutex<Box<Reactor>>>, data: u64, id: usize) -> Self {
        Task { id, reactor, data }
    }
}

// This is our `Future` implementation
impl Future for Task {
    type Output = usize;

    // Poll is the what drives the state machine forward and it's the only
    // method we'll need to call to drive futures to completion.
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {

        // We need to get access the reactor in our `poll` method so we acquire
        // a lock on that.
        let mut r = self.reactor.lock().unwrap();

        // First we check if the task is marked as ready
        if r.is_ready(self.id) {

            // If it's ready we set its state to `Finished`
            *r.tasks.get_mut(&self.id).unwrap() = TaskState::Finished;
            Poll::Ready(self.id)
        
        // If it isn't finished we check the map we have stored in our Reactor
        // over id's we have registered and see if it's there
        } else if r.tasks.contains_key(&self.id) {

            // This is important. The docs says that on multiple calls to poll,
            // only the Waker from the Context passed to the most recent call
            // should be scheduled to receive a wakeup. That's why we insert
            // this waker into the map (which will return the old one which will
            // get dropped) before we return `Pending`.
            r.tasks.insert(self.id, TaskState::NotReady(cx.waker().clone()));
            Poll::Pending
        } else {

            // If it's not ready, and not in the map it's a new task so we
            // register that with the Reactor and return `Pending`
            r.register(self.data, cx.waker().clone(), self.id);
            Poll::Pending
        }

        // Note that we're holding a lock on the `Mutex` which protects the
        // Reactor all the way until the end of this scope. This means that
        // even if our task were to complete immidiately, it will not be
        // able to call `wake` while we're in our `Poll` method.

        // Since we can make this guarantee, it's now the Executors job to
        // handle this possible race condition where `Wake` is called after
        // `poll` but before our thread goes to sleep.
    }
}
```
这大部分都是直截了当的。 令人困惑的部分是我们需要构建 Waker 的奇怪方式，但是由于我们已经从原始部分创建了我们自己的 trait 对象，这看起来很熟悉。 事实上，这更简单。

我们在这里使用一个Arc来传递一个引用计数的MyWaker的借用。 这是相当正常的，并且使得这个操作变得简单和安全。 克隆一个Waker只是增加一个计数。Drop一个Waker只是简单地减少一个计数. 

在我们这种特定场景下,我们选择不使用`Arc`. 而使用这种更低层次方式实现的Waker才可以允许我们这么做.

事实上，如果我们只使用 Arc，那么我们就没有理由费尽心思去创建自己的 vtable 和 RawWaker。 我们可以实现一个普通的trait。

幸运的是，将来在标准库中也可以实现这个功能。 目前[这个特性仍然在实验中](https://rust-lang-nursery.github.io/futures-api-docs/0.3.0-alpha.13/futures/task/trait.ArcWake.html)，但是我猜想在成熟之后，这个特性将会成为标准库的一部分。

我们选择在这里传入一个整个reactor的引用, 这不正常。 reactor通常是一个全局性的资源，让我们注册感兴趣的事而不需要传入一个引用.


 
>**为什么在一个Lib中使用park/unpark是一个坏主意**
>
>他很容易死锁,因为任何人都可以获得执行器所在线程的句柄,然后调用park/unpark.

> 1. 一个future可以在另一个不同的线程上unpark执行器线程
> 2. 我们的执行器认为数据准备好了,然后醒来去轮询这个`Future`
> 3. 当被轮询时,这个`Future`还没有准备好,但是恰在此时,`Reactor`收到事件,调用了`Wake()`来unpark我们的线程. 
> 4. 这可能发生在我们再次睡眠之前,因为这些操作完全是并行的.
> 5. 我们的reactor已经调用过`wake`,但是我们的线程仍然在睡眠,因为刚刚调用wake的时候,我们的线程是醒着的.
> 6. 我们发生了死锁,然后我们的程序停止工作.


> 有一种情况是，我们的线程可能会出现所谓的虚假唤醒(可能会出乎意料地发生) ，如果我们运气不好，这可能会导致同样的死锁
 

有几种更好的方案,比如:

- std::sync::CondVar
- crossbeam::sync::Parker

### Reactor

这是最后的冲刺阶段，并不完全与`Future`相关，但是我们需要它来让我们的例子运行起来。

由于大多数时候并发只有在与外部世界(或者至少是一些外围设备)进行交互时才有意义，因此我们需要一些东西来抽象这些异步的交互.

这就是reacotor的工作. 大多数时候你看到的reactor都是用[Mio](https://github.com/tokio-rs/mio)这个库. 它早多个平台上提供了非阻塞API和事件通知机制.

reactor通常会提供类似于TcpStream(或任何其他资源)的东西，只不过您用TcpStream来创建I/O请求,而用reactor来创建Future.

 

我们的示例任务是一个计时器，它只生成一个线程，并将其置于休眠状态，休眠时间为我们指定的秒数。 我们在这里创建的reactor将创建一个表示每个计时器的`leaf-future`。 作为回报，reactor接收到一个唤醒器，一旦任务完成reactor将调用这个唤醒器。

为了能够在浏览器中运行这里的代码，没有太多真正的I/O，我们可以假装这实际上代表了一些有用的I/O操作。

我们的reactor看起来像这样:

```rust
// This is a "fake" reactor. It does no real I/O, but that also makes our
// code possible to run in the book and in the playground
// The different states a task can have in this Reactor
enum TaskState {
    Ready,
    NotReady(Waker),
    Finished,
}

// This is a "fake" reactor. It does no real I/O, but that also makes our
// code possible to run in the book and in the playground
struct Reactor {

    // we need some way of registering a Task with the reactor. Normally this
    // would be an "interest" in an I/O event
    dispatcher: Sender<Event>,
    handle: Option<JoinHandle<()>>,

    // This is a list of tasks
    tasks: HashMap<usize, TaskState>,
}

// This represents the Events we can send to our reactor thread. In this
// example it's only a Timeout or a Close event.
#[derive(Debug)]
enum Event {
    Close,
    Timeout(u64, usize),
}

impl Reactor {

    // We choose to return an atomic reference counted, mutex protected, heap
    // allocated `Reactor`. Just to make it easy to explain... No, the reason
    // we do this is:
    //
    // 1. We know that only thread-safe reactors will be created.
    // 2. By heap allocating it we can obtain a reference to a stable address
    // that's not dependent on the stack frame of the function that called `new`
    fn new() -> Arc<Mutex<Box<Self>>> {
        let (tx, rx) = channel::<Event>();
        let reactor = Arc::new(Mutex::new(Box::new(Reactor {
            dispatcher: tx,
            handle: None,
            tasks: HashMap::new(),
        })));
        
        // Notice that we'll need to use `weak` reference here. If we don't,
        // our `Reactor` will not get `dropped` when our main thread is finished
        // since we're holding internal references to it.

        // Since we're collecting all `JoinHandles` from the threads we spawn
        // and make sure to join them we know that `Reactor` will be alive
        // longer than any reference held by the threads we spawn here.
        let reactor_clone = Arc::downgrade(&reactor);

        // This will be our Reactor-thread. The Reactor-thread will in our case
        // just spawn new threads which will serve as timers for us.
        let handle = thread::spawn(move || {
            let mut handles = vec![];

            // This simulates some I/O resource
            for event in rx {
                println!("REACTOR: {:?}", event);
                let reactor = reactor_clone.clone();
                match event {
                    Event::Close => break,
                    Event::Timeout(duration, id) => {

                        // We spawn a new thread that will serve as a timer
                        // and will call `wake` on the correct `Waker` once
                        // it's done.
                        let event_handle = thread::spawn(move || {
                            thread::sleep(Duration::from_secs(duration));
                            let reactor = reactor.upgrade().unwrap();
                            reactor.lock().map(|mut r| r.wake(id)).unwrap();
                        });
                        handles.push(event_handle);
                    }
                }
            }

            // This is important for us since we need to know that these
            // threads don't live longer than our Reactor-thread. Our
            // Reactor-thread will be joined when `Reactor` gets dropped.
            handles.into_iter().for_each(|handle| handle.join().unwrap());
        });
        reactor.lock().map(|mut r| r.handle = Some(handle)).unwrap();
        reactor
    }

    // The wake function will call wake on the waker for the task with the
    // corresponding id.
    fn wake(&mut self, id: usize) {
        self.tasks.get_mut(&id).map(|state| {

            // No matter what state the task was in we can safely set it
            // to ready at this point. This lets us get ownership over the
            // the data that was there before we replaced it.
            match mem::replace(state, TaskState::Ready) {
                TaskState::NotReady(waker) => waker.wake(),
                TaskState::Finished => panic!("Called 'wake' twice on task: {}", id),
                _ => unreachable!()
            }
        }).unwrap();
    }

    // Register a new task with the reactor. In this particular example
    // we panic if a task with the same id get's registered twice 
    fn register(&mut self, duration: u64, waker: Waker, id: usize) {
        if self.tasks.insert(id, TaskState::NotReady(waker)).is_some() {
            panic!("Tried to insert a task with id: '{}', twice!", id);
        }
        self.dispatcher.send(Event::Timeout(duration, id)).unwrap();
    }

    // We send a close event to the reactor so it closes down our reactor-thread
    fn close(&mut self) {
        self.dispatcher.send(Event::Close).unwrap();
    }

    // We simply checks if a task with this id is in the state `TaskState::Ready`
    fn is_ready(&self, id: usize) -> bool {
        self.tasks.get(&id).map(|state| match state {
            TaskState::Ready => true,
            _ => false,
        }).unwrap_or(false)
    }
}

impl Drop for Reactor {
    fn drop(&mut self) {
        self.handle.take().map(|h| h.join().unwrap()).unwrap();
    }
}

```

虽然代码量很大，但实际上我们只是产生了一个新线程，并让它休眠一段时间，这是我们在创建任务时指定的。


虽然代码量很大，但实际上我们只是产生了一个新线程，并让它休眠一段时间，这是我们在创建任务时指定的。

在最后一章中，我们在一个可编辑的窗口中有整整200行，你可以按照自己喜欢的方式进行编辑和修改。

```rust
fn main() {
    // This is just to make it easier for us to see when our Future was resolved
    let start = Instant::now();

    // Many runtimes create a glocal `reactor` we pass it as an argument
    let reactor = Reactor::new();

    // Since we'll share this between threads we wrap it in a 
    // atmically-refcounted- mutex.
    let reactor = Arc::new(Mutex::new(reactor));
    
    // We create two tasks:
    // - first parameter is the `reactor`
    // - the second is a timeout in seconds
    // - the third is an `id` to identify the task
    let future1 = Task::new(reactor.clone(), 1, 1);
    let future2 = Task::new(reactor.clone(), 2, 2);

    // an `async` block works the same way as an `async fn` in that it compiles
    // our code into a state machine, `yielding` at every `await` point.
    let fut1 = async {
        let val = future1.await;
        let dur = (Instant::now() - start).as_secs_f32();
        println!("Future got {} at time: {:.2}.", val, dur);
    };

    let fut2 = async {
        let val = future2.await;
        let dur = (Instant::now() - start).as_secs_f32();
        println!("Future got {} at time: {:.2}.", val, dur);
    };

    // Our executor can only run one and one future, this is pretty normal
    // though. You have a set of operations containing many futures that
    // ends up as a single future that drives them all to completion.
    let mainfut = async {
        fut1.await;
        fut2.await;
    };

    // This executor will block the main thread until the futures is resolved
    block_on(mainfut);

    // When we're done, we want to shut down our reactor thread so our program
    // ends nicely.
    reactor.lock().map(|mut r| r.close()).unwrap();
}

```

我添加了一个reactor感兴趣的事件的调试输出，这样我们可以观察到两件事:

1. `Waker`这个对象如何像前面我们讨论的trai对象
2. 事件以何种顺序向reactor注册感兴趣的信息


### Async/Await和并发Async/Await

Async 关键字可以用在 async fn (...)中的函数上，也可以用在 async { ... }中的块上。 两者都可以讲一个函数或者代码块转换成一个`Future`

这些`Future`是相当简单的。 想象一下几章前我们的生成器。 

每一个await就像一个yield,只不过不是生成一个值,而是生成Future,然后当轮询的时候返回响应的结果.

我们的`mainfut`包含两个`non-leaf-future`，它将在轮询中调用。`non-leaf-future`有一个`poll`方法, 这个方法简单的轮询他自己的内部Future,它内部的Future会被继续轮询,直到`leaf-future`返回`Ready`或者`Pending`.

就我们现在的例子来看，它并不比常规的同步代码好多少。 对于我们来说，如果需要在同一时间等待多个`Future`，我们需要`spawn`它们，以便执行器同时运行它们。

现在我们的例子返回如下结果:
```rust
Future got 1 at time: 1.00.
Future got 2 at time: 3.00.
```

```rust
Future got 1 at time: 1.00.
Future got 2 at time: 2.00.
```

> 请注意，这并不意味着它们需要并行运行。 它们可以并行运行，但没有要求。 请记住，我们正在等待一些外部资源，这样我们就可以在一个线程上发出许多这样的调用，并在事件发生时处理每个事件

现在，我将向您介绍一些更好的资源，以实现一个更好的执行器。 现在你应该已经对`Future`的概念有了一个很好的理解。

下一步应该是了解更高级的运行时是如何工作的，以及它们如何实现不同的运行 Futures 的方式。



如果我是你，我接下来就会读这篇文章，并试着把它应用到我们的例子中去。

 

我希望在阅读完这篇文章后，能够更容易地探索Future和异步，我真的希望你们能够继续深入探索。

别忘了最后一章的练习。

### 奖励部分-暂停线程的更好办法

正如我们在本章前面解释的那样，仅仅调用`thread::sleep` 并不足以实现一个合适的反应器。 你也可以使用类似[crossbeam::sync::Parker](https://docs.rs/crossbeam/0.7.3/crossbeam/sync/struct.Parker.html)中的Parker 这样的工具.

因为我们自己创建一个这样的Parker也不需要很多行代码，所以我们将展示如何通过使用 Condvar 和 Mutex 来解决这个问题。

我们自己的Parker:

```rust
#[derive(Default)]
struct Parker(Mutex<bool>, Condvar);

impl Parker {
    fn park(&self) {

        // We aquire a lock to the Mutex which protects our flag indicating if we
        // should resume execution or not.
        let mut resumable = self.0.lock().unwrap();

            // We put this in a loop since there is a chance we'll get woken, but
            // our flag hasn't changed. If that happens, we simply go back to sleep.
            while !*resumable {

                // We sleep until someone notifies us
                resumable = self.1.wait(resumable).unwrap();
            }

        // We immidiately set the condition to false, so that next time we call `park` we'll
        // go right to sleep.
        *resumable = false;
    }

    fn unpark(&self) {
        // We simply acquire a lock to our flag and sets the condition to `runnable` when we
        // get it.
        *self.0.lock().unwrap() = true;

        // We notify our `Condvar` so it wakes up and resumes.
        self.1.notify_one();
    }
}

```

在 Rust 中的 Condvar 被设计为与互斥对象一起工作。 通常，您会认为在我们进入休眠之前,`self.0.lock().unwrap()`不会释放锁, 这意味着我们的`unpark`永远获取不到锁,我们会陷入死锁。

使用`Condvar`我们可以避免这种情况，因为`Condvar`会消耗我们的锁，所以它会在我们睡觉的时候释放。
当我们再次恢复时，我们的`Condvar`会重新持有锁，这样我们就可以继续操作它。
这意味着我们需要对我们的执行器做一些非常细微的改变，比如:

```rust
fn block_on<F: Future>(mut future: F) -> F::Output {
    let parker = Arc::new(Parker::default()); // <--- NB!
    let mywaker = Arc::new(MyWaker { parker: parker.clone() }); <--- NB!
    let waker = mywaker_into_waker(Arc::into_raw(mywaker));
    let mut cx = Context::from_waker(&waker);
    
    // SAFETY: we shadow `future` so it can't be accessed again.
    let mut future = unsafe { Pin::new_unchecked(&mut future) }; 
    loop {
        match Future::poll(future.as_mut(), &mut cx) {
            Poll::Ready(val) => break val,
            Poll::Pending => parker.park(), // <--- NB!
        };
    }
}

```
我们需要像这样改变我们的唤醒器:

```rust
#[derive(Clone)]
struct MyWaker {
    parker: Arc<Parker>,
}

fn mywaker_wake(s: &MyWaker) {
    let waker_arc = unsafe { Arc::from_raw(s) };
    waker_arc.parker.unpark();
}

```
> 你可以查看由park/unpark引起的[微妙问题的连接](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=b2343661fe3d271c91c6977ab8e681d0). 你可以在[这里](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=bebef0f8a8ce6a9d0d32442cc8381595)查看我们最终的版本如何避免了这个问题.

