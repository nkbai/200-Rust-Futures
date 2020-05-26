##  背景资料

在我们深入研究 Futures in Rust 的细节之前，让我们快速了解一下处理并发编程的各种方法，以及每种方法的优缺点。

同时当涉及到并发性时,我们也会解释为什么这么做，这将使我们更容易深入理解Futures.

> 为了好玩，我在大多数示例中添加了一小段可运行代码。 如果你像我一样，事情会变得更有趣，也许你会看到一些你从未见过的东西。

### 线程

现在，实现这一点的一个方法就是让操作系统为我们处理好一切。 我们只需为每个要完成的任务生成一个新的操作系统线程，并像通常那样编写代码。

我们用来处理并发性的运行时就是操作系统本身。

优点:

- 简单
- 易用
- 在任务之间切换相当快
- 不需要付出即可得到并行支持
  
缺点:

- 操作系统级线程的堆栈相当大。 如果同时有许多任务等待(就像在负载很重的 web 服务器中那样) ，那么内存将很快耗尽
- 这涉及到很多系统调用。当任务数量很高时，这可能会非常昂贵
- 操作系统有很多事情需要处理。 它可能不会像你希望的那样快速地切换回线程
- 某些系统可能不支持线程
  
**在 Rust 中使用操作系统线程看起来像这样:**
```rust
use std::thread;

fn main() {
    println!("So we start the program here!");
    let t1 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(200));
        println!("We create tasks which gets run when they're finished!");
    });

    let t2 = thread::spawn(move || {
        thread::sleep(std::time::Duration::from_millis(100));
        println!("We can even chain callbacks...");
        let t3 = thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(50));
            println!("...like this!");
        });
        t3.join().unwrap();
    });
    println!("While our tasks are executing we can do other stuff here.");

    t1.join().unwrap();
    t2.join().unwrap();
}
```

操作系统线程肯定有一些相当大的优势。 这也是为什么所有这些讨论“异步”和并发性把线程摆在首位？

首先。 为了使计算机有效率，它们需要多任务处理。 一旦你开始深入研究(比如[操作系统是如何工作的](https://os.phil-opp.com/async-await/)) ，你就会发现并发无处不在。 这是我们做任何事情的基础。


其次，我们有网络。

Webservers 是关于I/O和处理小任务(请求)的。 当小任务的数量很大时，由于它们所需的内存和创建新线程所涉及的开销，就不适合今天的操作系统线程。

如果负载是可变的，那么问题就更大了，这意味着程序在任何时间点的当前任务数量都是不可预测的。 这就是为什么今天你会看到如此多的异步web框架和数据库驱动程序。

然而有大量的问题,标准的线程通常是正确的解决方案。 因此在使用异步库之前，请三思而后行。

现在，让我们来看看多任务处理的其他选项。 它们都有一个共同点，那就是它们实现了一种多任务处理的方式，即拥有一个“用户界面”运行时:

### 绿色线程(Green threads)

绿色线程使用与操作系统相同的机制，为每个任务创建一个线程，设置一个堆栈，保存 CPU 状态，并通过“上下文切换”从一个任务(线程)跳转到另一个任务(线程)。

我们将控制权交给调度程序(在这样的系统中，调度程序是运行时的核心部分) ，然后调度程序继续运行不同的任务。

Rust曾经支持绿色线程，但他们它达到1.0之前被删除了， 执行状态存储在每个栈中，因此在这样的解决方案中不需要`async`,`await`,`Futures` 或者`Pin`。

典型的流程是这样的:
1. 运行一些非阻塞代码
2. 对某些外部资源进行阻塞调用
3. 跳转到main”线程，该线程调度一个不同的线程来运行，并“跳转”到该栈中
4. 在新线程上运行一些非阻塞代码，直到新的阻塞调用或任务完成
5. “跳转”回到“main"线程 ，调度一个新线程，这个新线程的状态已经是`Ready`,然后跳转到该线程
   
这些“跳转”被称为上下文切换，当你阅读这篇文章的时候，你的操作系统每秒钟都会做很多次。

优点:

1. 栈大小可能需要增长,解决这个问题不容易,并且会有成本.[^go中的栈] 
    [^go中的栈]:  栈拷贝,指针等问题
2. 它不是一个零成本抽象(这也是Rust早期有绿色线程,后来删除的原因之一)
3.  如果您想要支持许多不同的平台，就很难正确实现

一个绿色线程的例子可以是这样的:
> 下面的例子是一个改编的例子，来自我之前写的一本[200行Rust说清绿色线程](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/)的 gitbook。 如果你想知道发生了什么，你会发现书中详细地解释了一切。 下面的代码非常不安全，只是为了展示一个真实的例子。 这绝不是为了展示“最佳实践”。 这样我们就能达成共识了。

```rust
 #![feature(asm, naked_functions)]
use std::ptr;

const DEFAULT_STACK_SIZE: usize = 1024 * 1024 * 2;
const MAX_THREADS: usize = 4;
static mut RUNTIME: usize = 0;

pub struct Runtime {
    threads: Vec<Thread>,
    current: usize,
}

  #[derive(PartialEq, Eq, Debug)]
enum State {
    Available,
    Running,
    Ready,
}

struct Thread {
    id: usize,
    stack: Vec<u8>,
    ctx: ThreadContext,
    state: State,
    task: Option<Box<dyn Fn()>>,
}

 #[derive(Debug, Default)]
 #[repr(C)]
struct ThreadContext {
    rsp: u64,
    r15: u64,
    r14: u64,
    r13: u64,
    r12: u64,
    rbx: u64,
    rbp: u64,
    thread_ptr: u64,
}

impl Thread {
    fn new(id: usize) -> Self {
        Thread {
            id,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Available,
            task: None,
        }
    }
}

impl Runtime {
    pub fn new() -> Self {
        let base_thread = Thread {
            id: 0,
            stack: vec![0_u8; DEFAULT_STACK_SIZE],
            ctx: ThreadContext::default(),
            state: State::Running,
            task: None,
        };

        let mut threads = vec![base_thread];
        threads[0].ctx.thread_ptr = &threads[0] as *const Thread as u64;
        let mut available_threads: Vec<Thread> = (1..MAX_THREADS).map(|i| Thread::new(i)).collect();
        threads.append(&mut available_threads);

        Runtime {
            threads,
            current: 0,
        }
    }

    pub fn init(&self) {
        unsafe {
            let r_ptr: *const Runtime = self;
            RUNTIME = r_ptr as usize;
        }
    }

    pub fn run(&mut self) -> ! {
        while self.t_yield() {}
        std::process::exit(0);
    }

    fn t_return(&mut self) {
        if self.current != 0 {
            self.threads[self.current].state = State::Available;
            self.t_yield();
        }
    }

    fn t_yield(&mut self) -> bool {
        let mut pos = self.current;
        while self.threads[pos].state != State::Ready {
            pos += 1;
            if pos == self.threads.len() {
                pos = 0;
            }
            if pos == self.current {
                return false;
            }
        }
        
        if self.threads[self.current].state != State::Available {
            self.threads[self.current].state = State::Ready;
        }

        self.threads[pos].state = State::Running;
        let old_pos = self.current;
        self.current = pos;

        unsafe {
            switch(&mut self.threads[old_pos].ctx, &self.threads[pos].ctx);
        }
        true
    }

    pub fn spawn<F: Fn() + 'static>(f: F){
        unsafe {
            let rt_ptr = RUNTIME as *mut Runtime;
            let available = (*rt_ptr)
                .threads
                .iter_mut()
                .find(|t| t.state == State::Available)
                .expect("no available thread.");
                
            let size = available.stack.len();
            let s_ptr = available.stack.as_mut_ptr();
            available.task = Some(Box::new(f));
            available.ctx.thread_ptr = available as *const Thread as u64;
            ptr::write(s_ptr.offset((size - 8) as isize) as *mut u64, guard as u64);
            ptr::write(s_ptr.offset((size - 16) as isize) as *mut u64, call as u64);
            available.ctx.rsp = s_ptr.offset((size - 16) as isize) as u64;
            available.state = State::Ready;
        }
    }
}

fn call(thread: u64) {
    let thread = unsafe { &*(thread as *const Thread) };
    if let Some(f) = &thread.task {
        f();
    }
}

 #[naked]
fn guard() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        let rt = &mut *rt_ptr;
        println!("THREAD {} FINISHED.", rt.threads[rt.current].id);
        rt.t_return();
    };
}

pub fn yield_thread() {
    unsafe {
        let rt_ptr = RUNTIME as *mut Runtime;
        (*rt_ptr).t_yield();
    };
}

 #[naked]
 #[inline(never)]
unsafe fn switch(old: *mut ThreadContext, new: *const ThreadContext) {
    asm!("
        mov     %rsp, 0x00($0)
        mov     %r15, 0x08($0)
        mov     %r14, 0x10($0)
        mov     %r13, 0x18($0)
        mov     %r12, 0x20($0)
        mov     %rbx, 0x28($0)
        mov     %rbp, 0x30($0)

        mov     0x00($1), %rsp
        mov     0x08($1), %r15
        mov     0x10($1), %r14
        mov     0x18($1), %r13
        mov     0x20($1), %r12
        mov     0x28($1), %rbx
        mov     0x30($1), %rbp
        mov     0x38($1), %rdi
        ret
        "
    :
    : "r"(old), "r"(new)
    :
    : "alignstack"
    );
}
 #[cfg(not(windows))]
fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    Runtime::spawn(|| {
        println!("I haven't implemented a timer in this example.");
        yield_thread();
        println!("Finally, notice how the tasks are executed concurrently.");
    });
    Runtime::spawn(|| {
        println!("But we can still nest tasks...");
        Runtime::spawn(|| {
            println!("...like this!");
        })
    });
    runtime.run();
}
 #[cfg(windows)]
fn main() {
    let mut runtime = Runtime::new();
    runtime.init();
    Runtime::spawn(|| {
        println!("I haven't implemented a timer in this example.");
        yield_thread();
        println!("Finally, notice how the tasks are executed concurrently.");
    });
    Runtime::spawn(|| {
        println!("But we can still nest tasks...");
        Runtime::spawn(|| {
            println!("...like this!");
        })
    });
    runtime.run();
}

```


还在坚持阅读本书？ 很好。 如果上面的代码很难理解，不要感到沮丧。 如果不是我自己写的，我可能也会有同样的感觉。 你随时可以回去读,稍后我还会解释。

### 基于回调的方法

你可能已经知道我们接下来要谈论Javascript，我想大多数人都知道。

> 如果你接触过 Javascript 回调会让你更早患上 PTSD，那么现在闭上眼睛，向下滚动2-3秒。 你会在那里找到一个链接，带你到安全的地方。

基于回调的方法背后的整个思想就是保存一个指向一组指令的指针，这些指令我们希望以后在以后需要的时候运行。 针对Rust，这将是一个闭包。 在下面的示例中，我们将此信息保存在`HashMap`中，但这并不是唯一的选项。

不涉及线程作为实现并发性的主要方法的基本思想是其余方法的共同点。 包括我们很快就会讲到的 Rust 今天使用的那个。

优点:
1. 大多数语言中易于实现
2. 没有上下文切换
3. 相对较低的内存开销(在大多数情况下)

缺点:

1. 每个任务都必须保存它以后需要的状态，内存使用量将随着一系列计算中回调的数量线性增长
2. 很难理解，很多人已经知道这就是“回调地狱”
3. 这是一种非常不同的编写程序的方式，需要大量的重写才能从“正常”的程序流转变为使用“基于回调”的程序流
4. 在 Rust 使用这种方法时，任务之间的状态共享是一个难题，因为它的所有权模型
   
一个极其简单的基于回调方法的例子是:

```rust
fn program_main() {
    println!("So we start the program here!");
    set_timeout(200, || {
        println!("We create tasks with a callback that runs once the task finished!");
    });
    set_timeout(100, || {
        println!("We can even chain sub-tasks...");
        set_timeout(50, || {
            println!("...like this!");
        })
    });
    println!("While our tasks are executing we can do other stuff instead of waiting.");
}

fn main() {
    RT.with(|rt| rt.run(program_main));
}

use std::sync::mpsc::{channel, Receiver, Sender};
use std::{cell::RefCell, collections::HashMap, thread};

thread_local! {
    static RT: Runtime = Runtime::new();
}

struct Runtime {
    callbacks: RefCell<HashMap<usize, Box<dyn FnOnce() -> ()>>>,
    next_id: RefCell<usize>,
    evt_sender: Sender<usize>,
    evt_reciever: Receiver<usize>,
}

fn set_timeout(ms: u64, cb: impl FnOnce() + 'static) {
    RT.with(|rt| {
        let id = *rt.next_id.borrow();
        *rt.next_id.borrow_mut() += 1;
        rt.callbacks.borrow_mut().insert(id, Box::new(cb));
        let evt_sender = rt.evt_sender.clone();
        thread::spawn(move || {
            thread::sleep(std::time::Duration::from_millis(ms));
            evt_sender.send(id).unwrap();
        });
    });
}

impl Runtime {
    fn new() -> Self {
        let (evt_sender, evt_reciever) = channel();
        Runtime {
            callbacks: RefCell::new(HashMap::new()),
            next_id: RefCell::new(1),
            evt_sender,
            evt_reciever,
        }
    }

    fn run(&self, program: fn()) {
        program();
        for evt_id in &self.evt_reciever {
            let cb = self.callbacks.borrow_mut().remove(&evt_id).unwrap();
            cb();
            if self.callbacks.borrow().is_empty() {
                break;
            }
        }
    }
}
```

我们保持这种超级简单的方法，您可能想知道这种方法和使用 OS 线程直接将回调传递给 OS 线程的方法之间有什么区别。
不同之处在于，回调是在同一个线程上运行的。 这个例子中,我们创建的 OS 线程基本上只是用作计时器，但可以表示任何类型的我们将不得不等待的资源。

### 从回调到承诺 (promises)

你现在可能会想，我们什么时候才能谈论未来？

好吧，我们就快到了。你看，`promises`、`futures`和其他延迟计算的名称经常被交替使用。

它们之间有形式上的区别，但是我们在这里不会涉及，但是值得解释一下`promises`，因为它们被广泛使用在 Javascript 中，并且与 Rusts Futures 有很多共同之处。

首先，许多语言都有`promises`的概念，但我将在下面的例子中使用来自 Javascript 的概念。

承诺是解决回调带来的复杂性的一种方法。

比如,下面的例子:
```js
setTimer(200, () => {
  setTimer(100, () => {
    setTimer(50, () => {
      console.log("I'm the last one");
    });
  });
});
```

可以替换为promise:

```js
function timer(ms) {
    return new Promise((resolve) => setTimeout(resolve, ms))
}

timer(200)
.then(() => return timer(100))
.then(() => return timer(50))
.then(() => console.log("I'm the last one));

```

深入原理可以看到变化更为显著。 您可以看到，promises 返回的状态机可以处于以下三种状态之一: `pending`、 `fulfilled` 或 `rejected`。

当我们在上面的例子中调用`timer (200)`时，我们得到一个状态`pending`的承诺。

由于承诺被重写为状态机，它们还提供了一种更好的语法，允许我们像下面这样编写最后一个示例:

```js
async function run() {
    await timer(200);
    await timer(100);
    await timer(50);
    console.log("I'm the last one");
}
```

可以将 run 函数视为一个由几个子任务组成的可执行任务。 在每个`await`点上，它都将控制权交给调度程序(在本例中是众所周知的 Javascript 事件循环)。

一旦其中一个子任务将状态更改为`fulfilled`或`rejected`，则计划继续执行下一步。

从语法上讲，Rusts Futures 0.1很像上面的承诺示例，Rusts Futures 0.3很像我们上一个示例中的 async / await。

这也是与 Rusts Futures 相似的地方。 我们这样做的原因是通过上面的介绍,更加深刻的理解Rust的Futures。

> 为了避免以后的混淆: 有一点你应该知道。 Java script的承诺是立即执行(early evaluated)的。 这意味着一旦它被创建，它就开始运行一个任务。 与此相反,Rust的Futures是延迟执行(lazy evaluated)。 除非轮询(poll)一次,否则什么事都不会发生。


