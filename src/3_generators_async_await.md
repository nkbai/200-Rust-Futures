##  生成器和async/await

### 概述

1. 理解 async / await 语法在底层是如何工作的
2. 亲眼目睹(See first hand)我们为什么需要Pin
3. 理解是什么让 Rusts 异步模型的内存效率非常高

生成器的动机可以在 [RFC#2033](https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md)中找到。 它写得非常好，我建议您通读它(它谈论async/await的内容和谈论生成器的内容一样多)。

### 为什么要学习生成器

generators/yield和 async/await 非常相似，一旦理解了其中一个，就应该能够理解另一个。

对我来说，使用Generators而不是 Futures 来提供可运行的和简短的示例要容易得多，这需要我们现在引入很多概念，稍后我们将介绍这些概念，以便展示示例。

Async/await 的工作方式类似于生成器，但它不返回生成器，而是返回一个实现 Future trait 的特殊对象。

一个小小的好处是，在本章的最后，你将有一个很好的关于生成器和 async / await 的介绍。

基本上，在设计 Rust 如何处理并发时，主要讨论了三个选项:
1. Green Thread. 
2. 使用组合符(Using combinators.)
3. Generator, 没有专门的栈

我们在背景信息中覆盖了绿色线程，所以我们不会在这里重复。 我们将集中在各种各样的无堆栈协同程序,这也就是Rust正在使用的.

### 组合子(Combinators)

在`Futures 0.1`中使用组合子.如果你曾经是用过Javascript中的`Promises`,那么你已经比较熟悉combinators了. 在Rust中,他们看起来如下:
```rust
let future = Connection::connect(conn_str).and_then(|conn| {
    conn.query("somerequest").map(|row|{
        SomeStruct::from(row)
    }).collect::<Vec<SomeStruct>>()
});

let rows: Result<Vec<SomeStruct>, SomeLibraryError> = block_on(future);
```

使用这个技巧主要有三个缺点:
1. 错误消息可能会冗长并且难懂
2. 不是最佳的内存使用(浪费内存)
3. Rust中不允许跨组合子借用.

其中第三点是这种方式的主要缺点.

不允许跨组合子借用，结果是非常不符合人体工程学的.为了完成某些任务，需要额外的内存分配或者复制，这很低效。

内存占用高的原因是，这基本上是一种基于回调的方法，其中每个闭包存储计算所需的所有数据。 这意味着，随着我们将它们链接起来，存储所需状态所需的内存会随着每一步的增加而增加。

### 无栈协程/生成器
这就是今天 Rust 使用的模型，它有几个显著的优点:
1. 使用 async/await 作为关键字，可以很容易地将普通的Rust代码转换为无堆栈的协程(甚至可以使用宏来完成)
2. 不需要上下文切换与保存恢复CPU状态
3. 不需要处理的动态栈分配
4. 内存效率高
5. 允许我们块暂停点(suspension)借用 **这是啥意思啊**

与Futures 0.1不一样，使用async/ await 我们可以这样做:

```rust
async fn myfn() {
    let text = String::from("Hello world");
    let borrowed = &text[0..5];
    somefuture.await;
    println!("{}", borrowed);
}
```
Rust中的异步使用生成器实现.  因此为了理解异步是如何工作的，我们首先需要理解生成器。 在Rust中，生成器被实现为状态机。

一个计算链的内存占用是由占用空间最大的那个步骤定义的。

这意味着在计算链中添加步骤可能根本不需要增加任何内存，这也是为什么Futures和 Async 在 Rust 中的开销很小的原因之一。


###  生成器是如何工作的


在今天的 Nightly Rust 中，你可以使用关键词 yield。 在闭包中使用这个关键字，将其转换为生成器。 在介绍Pin之前，闭包是这样的:

```rust
 #![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};

fn main() {
    let a: i32 = 4;
    let mut gen = move || {
        println!("Hello");
        yield a * 2;
        println!("world!");
    };

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
```

早些时候，在人们对 Pin 的设计达成共识之前，编译完代码看起来类似于这样:

```rust
fn main() {
    let mut gen = GeneratorA::start(4);

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}

// If you've ever wondered why the parameters are called Y and R the naming from
// the original rfc most likely holds the answer
enum GeneratorState<Y, R> {
    Yielded(Y),  // originally called `Yield(Y)`
    Complete(R), // originally called `Return(R)`
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter(i32),
    Yield1(i32),
    Exit,
}

impl GeneratorA {
    fn start(a1: i32) -> Self {
        GeneratorA::Enter(a1)
    }
}

impl Generator for GeneratorA {
    type Yield = i32;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(self, GeneratorA::Exit) {
            GeneratorA::Enter(a1) => {

          /*----code before yield----*/
                println!("Hello");
                let a = a1 * 2;

                *self = GeneratorA::Yield1(a);
                GeneratorState::Yielded(a)
            }

            GeneratorA::Yield1(_) => {
          /*-----code after yield-----*/
                println!("world!");

                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}

```

关键词yield首先在[RFC#1823](https://github.com/rust-lang/rfcs/pull/1823)和 [RFC#1832](https://github.com/rust-lang/rfcs/pull/1832)中讨论。

既然您知道了现实中的 yield 关键字会将代码重写为状态机，那么您还将了解await 如何工作的,他们非常相似.

上述简单的状态机中有一些限制,当跨yield发生借用的时候会发生什么呢?

我们可以禁止这样做，但async/await 语法的主要设计目标之一就是允许这样做。 这些类型的借用是不可能使用`Futures 0.1`，所以我们不能让这个限制存在。

与其在理论上讨论它，不如让我们来看看一些代码。

> 我们将使用目前 Rust 中使用的状态机的优化版本。 更深入的解释见 Tyler Mandry 的文章: [Rust 如何优化async/await](https://tmandry.gitlab.io/blog/posts/optimizing-await-1/)


```rust
let mut generator = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
```

我们将手工编写一些版本的状态机，这些状态机表示生成器定义的状态机。

在每个示例中，我们都是“手动”逐步完成每个步骤，因此它看起来非常陌生。 我们可以添加一些语法糖，比如为我们的生成器实现 Iterator trait，这样我们就可以这样做:

```rust
while let Some(val) = generator.next() {
    println!("{}", val);
}
```

这是一个相当微不足道的改变，但是这一章已经变得很长了。我们继续前进的时候，请牢牢记住这点。

现在，我们的重写状态机在这个示例中看起来是什么样子的？
```rust

 #![allow(unused_variables)]
fn main() {
enum GeneratorState<Y, R> {
    Yielded(Y), 
    Complete(R),
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: &String, // uh, what lifetime should this have?
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        match std::mem::replace(self, GeneratorA::Exit) {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow; // <--- NB!
                let res = borrowed.len();

                *self = GeneratorA::Yield1 {to_borrow, borrowed};
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {to_borrow, borrowed} => {
                println!("Hello {}", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
}
```

如果你试图编译这个，你会得到一个错误。

字符串的生命周期是什么。 这和Self的生命周期是不一样的。 它不是静态的。 事实证明，我们不可能用Rusts语法来描述这个生命周期，这意味着，为了使这个工作成功，我们必须让编译器知道，我们自己正确地控制了它。

这意味着必须借助unsafe。

让我们尝试编写一个使用unsafe的实现。 正如您将看到的，我们最终将使用一个自引用结构, 也就是将引用保存在自身中的结构体。


正如您所注意到的，这个编译器编译得很好！

```rust

 #![allow(unused_variables)]
fn main() {
enum GeneratorState<Y, R> {
    Yielded(Y),  
    Complete(R), 
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String, // NB! This is now a raw pointer!
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}
impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
            match self {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                
                // NB! And we set the pointer to reference the to_borrow string here
                if let GeneratorA::Yield1 {to_borrow, borrowed} = self {
                    *borrowed = to_borrow;
                }
               
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
}
```
请记住，我们的例子是我们生成的生成器，它的原始文件像这样:
```rust
let mut gen = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
```
下面是我们如何运行这个状态机的示例，正如您所看到的，它完成了我们所期望的任务。 但这仍然存在一个巨大的问题:


```rust
pub fn main() {
    let mut gen = GeneratorA::start();
    let mut gen2 = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Yielded(n) = gen2.resume() {
        println!("Got value {}", n);
    }

    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
enum GeneratorState<Y, R> {
    Yielded(Y),  
    Complete(R), 
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String,
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}
impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
            match self {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                
                // We set the self-reference here
                if let GeneratorA::Yield1 {to_borrow, borrowed} = self {
                    *borrowed = to_borrow;
                }
               
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```
问题在于，如果在Safe Rust代码中,我们这样做:
```rust
 #![feature(never_type)] // Force nightly compiler to be used in playground
// by betting on it's true that this type is named after it's stabilization date...
pub fn main() {
    let mut gen = GeneratorA::start();
    let mut gen2 = GeneratorA::start();

    if let GeneratorState::Yielded(n) = gen.resume() {
        println!("Got value {}", n);
    }

    std::mem::swap(&mut gen, &mut gen2); // <--- Big problem!

    if let GeneratorState::Yielded(n) = gen2.resume() {
        println!("Got value {}", n);
    }

    // This would now start gen2 since we swapped them.
    if let GeneratorState::Complete(()) = gen.resume() {
        ()
    };
}
enum GeneratorState<Y, R> {
    Yielded(Y),  
    Complete(R), 
}

trait Generator {
    type Yield;
    type Return;
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return>;
}

enum GeneratorA {
    Enter,
    Yield1 {
        to_borrow: String,
        borrowed: *const String,
    },
    Exit,
}

impl GeneratorA {
    fn start() -> Self {
        GeneratorA::Enter
    }
}
impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(&mut self) -> GeneratorState<Self::Yield, Self::Return> {
            match self {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *self = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};
                
                // We set the self-reference here
                if let GeneratorA::Yield1 {to_borrow, borrowed} = self {
                    *borrowed = to_borrow;
                }
               
                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *self = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

运行代码并比较结果。你看到问题了吗？

等等? “Hello”怎么了? 为什么我们的代码出错了？

事实证明，虽然上面的例子编译得很好，但是我们在使用安全Rust时将这个API的使用者暴露在可能的内存未定义行为和其他内存错误中。 这是个大问题！


实际上，我已经强制上面的代码使用编译器的夜间版本。 如果您在playground上运行上面的示例，您将看到它在当前稳定状态(1.42.0)上运行时没有panic，但在当前夜间状态(1.44.0)上panic。 太可怕了！

我们将在下一章用一个稍微简单一点的例子来解释这里发生了什么，我们将使用 Pin 来修复我们的生成器，所以不用担心，您将看到到底出了什么问题，看看 Pin 如何能够帮助我们在一秒钟内安全地处理自引用类型。

在我们详细解释这个问题之前，让我们通过了解生成器和 async 关键字之间的关系来结束本章。

### 异步和生成器
Futures 在Rust中被实现为状态机，就像生成器是状态机一样。

您可能已经注意到异步块中使用的语法和生成器中使用的语法的相似之处:

```rust
let mut gen = move || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };

```

比较一下异步块的类似例子:

```rust
let mut fut = async {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        SomeResource::some_task().await;
        println!("{} world!", borrowed);
    };
```

不同之处在于，Futures 的状态与 Generator 的状态不同。

异步块将返回一个 Future 而不是 Generator，但是 Future 的工作方式和 Generator 的内部工作方式是相似的。

我们不调用`Generator::resume`，而是调用 `Future::poll`，并且不返回 generated 或 Complete，而是返回 Pending 或 Ready。 Future中的每一个await就像生成器中的一个yield。

你看到他们现在是怎么联系起来的了吗？


这就是为什么理解了生成器如何工作以及他需要面对的挑战,也就理解了Future如何工作以及它需要面对的挑战。

跨yield/await的借用就是这样.

### 奖励部分-正在使用的自引用生成器

感谢[ PR#45337 ](https://github.com/rust-lang/rust/pull/45337/files),你可以在nightly版本中使用static关键字运行上面的例子. 你可以试试:

> 要注意的是，API可能会发生改变。 在我撰写本书时，生成器API有一个更改，添加了对“ resume”参数的支持，以便传递到生成器闭包中。
> 可以关注[RFC#033](https://github.com/rust-lang/rfcs/blob/master/text/2033-experimental-coroutines.md)的相关[问题#4312](https://github.com/rust-lang/rust/issues/43122)的进展。

```rust
 #![feature(generators, generator_trait)]
use std::ops::{Generator, GeneratorState};


pub fn main() {
    let gen1 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };
    
    let gen2 = static || {
        let to_borrow = String::from("Hello");
        let borrowed = &to_borrow;
        yield borrowed.len();
        println!("{} world!", borrowed);
    };

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume(()) {
        println!("Gen1 got value {}", n);
    }
    
    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume(()) {
        println!("Gen2 got value {}", n);
    };

    let _ = pinned1.as_mut().resume(());
    let _ = pinned2.as_mut().resume(());
}
```
