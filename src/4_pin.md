##   Pin

### 概述
> 译者注: Pin是在使用Future时一个非常重要的概念,我的理解是: 通过使用Pin,让用户无法安全的获取到`&mut T`,进而无法进行上述例子中的swap. 如果你觉得你的和这个struct没有自引用的问题,你可以自己实现UnPin. 

1.  了解如何使用Pin以及当你自己实现`Future`的时候为什么需要Pin
2. 理解如何让自引用类型被安全的使用
3. 理解跨'await`借用是如何实现的
4. 制定一套实用的规则来帮助你使用Pin

Pin是在[RFC#2349](https://github.com/rust-lang/rfcs/blob/master/text/2349-pin.md)中被提出的.

让我们直接了当的说吧,Pin是这一系列概念中很难一开始就搞明白的,但是一旦你理解了其心智模型,就会觉得非常容易理解.


### 定义

Pin只与指针有关,在Rust中引用也是指针.

Pin有`Pin`类型和`Unpin`标记组成(UnPin是Rust中为数不多的几个auto trait). Pin存在的目的就是为了让那些实现了`!UnPin`的类型遵守特定的规则.


是的，你是对的，这里是双重否定`!Unpin` 的意思是“not-un-pin”。


> 这个命名方案是 Rusts 的安全特性之一，它故意测试您是否因为太累而无法安全地使用这个标记来实现类型。 如果你因为`UnPin`开始感到困惑，或者甚至生气，那么你就应该这样做！ 是时候放下工作，以全新的心态重新开始明天的生活了，这是一个好兆头。

更严肃地说，我认为有必要提到，选择这些名字是有正当理由的。 命名并不容易，我曾经考虑过在这本书中重命名 `Unpin` 和`!UnPin` ,使他们更容易理解。

然而，一位经验丰富的Rust社区成员让我相信，当简单地给这些标记起不同的名字时，有太多的细微差别和边缘情况需要考虑，而这些很容易被忽略，我相信我们将不得不习惯它们并按原样使用它们。

如果你愿意，你可以从[内部讨论](https://internals.rust-lang.org/t/naming-pin-anchor-move/6864)中读到一些讨论。

### Pinning和自引用结构
让我们从上一章(生成器那一章)停止的地方开始，通过使用一些比状态机更容易推理的自引用结构，使我们在生成器中看到的使用自引用结构的问题变得简单得多:


现在我们的例子是这样的:

```rust
use std::pin::Pin;

  #[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
}

impl Test {
    fn new(txt: &str) -> Self {
        let a = String::from(txt);
        Test {
            a,
            b: std::ptr::null(),
        }
    }

    fn init(&mut self) {
        let self_ref: *const String = &self.a;
        self.b = self_ref;
    }
    
    fn a(&self) -> &str {
        &self.a
    } 
    
    fn b(&self) -> &String {
        unsafe {&*(self.b)}
    }
}
```

让我们来回顾一下这个例子，因为我们将在本章的其余部分使用它。

我们有一个自引用结构体`Test`。 Test需要创建一个init方法，这个方法很奇怪，但是为了尽可能简短，我们需要这个方法。

Test 提供了两种方法来获取字段 a 和 b 值的引用。 因为 b 是 a 的参考，所以我们把它存储为一个指针，因为 Rust 的借用规则不允许我们定义这个生命周期。

现在，让我们用这个例子来详细解释我们遇到的问题:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

在main函数中,我们首先实例化Test的两个实例,然后输出test1和test2各字段的值,结果如我们所料:

```
a: test1, b: test1
a: test2, b: test2
```
让我们看看，如果我们将存储在 test1指向的内存位置的数据与存储在 test2指向的内存位置的数据进行交换，会发生什么情况，反之亦然。

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```

我们可能会认为会打印两边test1,比如:

```rust
a: test1, b: test1
a: test1, b: test1
```

但是实际上我们得到的是:
```
a: test1, b: test1
a: test1, b: test2
```

指向 test2.b 的指针仍然指向test1内部的旧位置。 该结构不再是自引用的，它保存指向不同对象中的字段的指针。 这意味着我们不能再依赖test2.b的生存期与test2的生存期绑定在一起。

如果你仍然不相信，这至少可以说服你:

```rust
fn main() {
    let mut test1 = Test::new("test1");
    test1.init();
    let mut test2 = Test::new("test2");
    test2.init();

    println!("a: {}, b: {}", test1.a(), test1.b());
    std::mem::swap(&mut test1, &mut test2);
    test1.a = "I've totally changed now!".to_string();
    println!("a: {}, b: {}", test2.a(), test2.b());

}
```
这是不应该发生的。 目前还没有严重的错误，但是您可以想象，使用这些代码很容易创建严重的错误。

我创建了一个图表来帮助可视化正在发生的事情:

![](swap_problem.jpg)
图1: 交换前后

正如你看到的,这不是我们想要的结果. 这很容易导致段错误,也很容易导致其他意想不到的未知行为以及失败.

### 固定在栈上

现在，我们可以通过使用`Pin`来解决这个问题。 让我们来看看我们的例子是什么样的:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

 #[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}


impl Test {
    fn new(txt: &str) -> Self {
        let a = String::from(txt);
        Test {
            a,
            b: std::ptr::null(),
            // This makes our type `!Unpin`
            _marker: PhantomPinned,
        }
    }
    fn init<'a>(self: Pin<&'a mut Self>) {
        let self_ptr: *const String = &self.a;
        let this = unsafe { self.get_unchecked_mut() };
        this.b = self_ptr;
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}
```

现在，我们在这里所做的就是固定到一个栈地址。如果我们的类型实现了`!UnPin`，那么它将总是`unsafe`。

我们在这里使用相同的技巧，包括需要 init。 如果我们想要解决这个问题并让用户避免`unsafe`，我们需要将数据钉在堆上，我们马上就会展示这一点。

让我们看看如果我们现在运行我们的例子会发生什么:

```rust
pub fn main() {
    // test1 is safe to move before we initialize it
    let mut test1 = Test::new("test1");
    // Notice how we shadow `test1` to prevent it from beeing accessed again
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());
     
    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

现在，如果我们尝试使用上次使我们陷入麻烦的问题，您将得到一个编译错误。
```rust
pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test1 = unsafe { Pin::new_unchecked(&mut test1) };
    Test::init(test1.as_mut());
     
    let mut test2 = Test::new("test2");
    let mut test2 = unsafe { Pin::new_unchecked(&mut test2) };
    Test::init(test2.as_mut());

    println!("a: {}, b: {}", Test::a(test1.as_ref()), Test::b(test1.as_ref()));
    std::mem::swap(test1.as_mut(), test2.as_mut());
    println!("a: {}, b: {}", Test::a(test2.as_ref()), Test::b(test2.as_ref()));
}
```

正如您从运行代码所得到的错误中看到的那样，类型系统阻止我们交换固定指针。

> 需要注意的是，栈pinning总是依赖于我们所在的当前栈帧，因此我们不能在一个栈帧中创建一个自引用对象并返回它，因为任何指向“self”的指针都是无效的。
> 如果你把一个值固定在一个栈上，这也会让你承担很多责任。 一个很容易犯的错误是，忘记对原始变量进行阴影处理，因为这样可以在初始化后drop固定的指针并访问原来的值:
```rust
fn main() {
   let mut test1 = Test::new("test1");
   let mut test1_pin = unsafe { Pin::new_unchecked(&mut test1) };
   Test::init(test1_pin.as_mut());
   drop(test1_pin);
   
   let mut test2 = Test::new("test2");
   mem::swap(&mut test1, &mut test2);
   println!("Not self referential anymore: {:?}", test1.b);
}
```

### 固定在堆上


为了完整性，让我们删除一些不安全的内容，通过以堆分配为代价来消除`init`方法。 固定到堆是安全的，这样用户不需要实现任何不安全的代码:

```rust
use std::pin::Pin;
use std::marker::PhantomPinned;

 #[derive(Debug)]
struct Test {
    a: String,
    b: *const String,
    _marker: PhantomPinned,
}

impl Test {
    fn new(txt: &str) -> Pin<Box<Self>> {
        let a = String::from(txt);
        let t = Test {
            a,
            b: std::ptr::null(),
            _marker: PhantomPinned,
        };
        let mut boxed = Box::pin(t);
        let self_ptr: *const String = &boxed.as_ref().a;
        unsafe { boxed.as_mut().get_unchecked_mut().b = self_ptr };

        boxed
    }

    fn a<'a>(self: Pin<&'a Self>) -> &'a str {
        &self.get_ref().a
    }

    fn b<'a>(self: Pin<&'a Self>) -> &'a String {
        unsafe { &*(self.b) }
    }
}

pub fn main() {
    let mut test1 = Test::new("test1");
    let mut test2 = Test::new("test2");

    println!("a: {}, b: {}",test1.as_ref().a(), test1.as_ref().b());
    println!("a: {}, b: {}",test2.as_ref().a(), test2.as_ref().b());
}
```

事实上就算是`!Unpin`有意义,固定一个堆分配的值也是安全的。 一旦在堆上分配了数据，它就会有一个稳定的地址。

作为 API 的用户，我们不需要特别注意并确保自引用指针保持有效。

也有一些方法能够对固定栈上提供一些安全保证，但是现在我们使用[pin_project](https://docs.rs/pin-project/)这个包来实现这一点。

### Pinning的一些实用规则

1. 针对`T:UnPin`(这是默认值),`Pin<'a,T>`完全定价与`&'a mut T`. 换句话说: `UnPin`意味着这个类型即使在固定时也可以移动，所以Pin对这个类型没有影响。
2. 针对`T:!UnPin`,从`Pin< T>`获取到`&mut T`,则必须使用unsafe. 换句话说,`!Unpin`能够阻止API的使用者移动T,除非他写出unsafe的代码.
3. Pinning对于内存分配没有什么特别的作用，比如将其放入某个“只读”内存或任何奇特的内存中。 它只使用类型系统来防止对该值进行某些操作。
4. 大多数标准库类型实现 Unpin。 这同样适用于你在 Rust 中遇到的大多数“正常”类型。 `Future`和`Generators`是两个例外。
5. Pin的主要用途就是自引用类型,Rust语言的所有这些调整就是为了允许这个. 这个API中仍然有一些问题需要探讨.
6. `!UnPin`这些类型的实现很有可能是不安全的. 在这种类型被钉住后移动它可能会导致程序崩溃。 在撰写本书时，创建和读取自引用结构的字段仍然需要不安全的方法(唯一的方法是创建一个包含指向自身的原始指针的结构)。
7. 当使用nightly版本时,你可以在一个使用特性标记在一个类型上添加`!UnPin`. 当使用stable版本时,可以将std: : marker: : PhantomPinned 添加到类型上。
8. 你既可以固定一个栈上的对象也可以固定一个堆上的对象.
9. 将一个`!UnPin`的指向栈上的指针固定需要unsafe.
10. 将一个`!UnPin`的指向堆上的指针固定,不需要unsafe,可以直接使用`Box::Pin`.

> 不安全的代码并不意味着它真的“unsafe” ，它只是减轻了通常从编译器得到的保证。 一个不安全的实现可能是完全安全的，但是您没有编译器保证的安全网。

### 映射/结构体的固定

简而言之，投影是一个编程语言术语。 Mystruct.field1是一个投影。 结构体的固定是在每一个字段上使用Pin。 这里有一些注意事项，您通常不会看到，因此我参考相关文档。

### Pin和Drop

Pin保证从值被固定到被删除的那一刻起一直存在。 而在Drop实现中，您需要一个可变的 self 引用，这意味着在针对固定类型实现 Drop 时必须格外小心。

### 把它们放在一起

当我们实现自己的`Futures`的时候,这正是我们要做的，我们很快就完成了。

### 奖励部分

**修复我们实现的自引用生成器以及学习更多的关于Pin的知识.**

但是现在，让我们使用 Pin 来防止这个问题。 我一直在评论，以便更容易地发现和理解我们需要做出的改变。
```rust
 #![feature(optin_builtin_traits, negative_impls)] // needed to implement `!Unpin`
use std::pin::Pin;

pub fn main() {
    let gen1 = GeneratorA::start();
    let gen2 = GeneratorA::start();
    // Before we pin the pointers, this is safe to do
    // std::mem::swap(&mut gen, &mut gen2);

    // constructing a `Pin::new()` on a type which does not implement `Unpin` is
    // unsafe. A value pinned to heap can be constructed while staying in safe
    // Rust so we can use that to avoid unsafe. You can also use crates like
    // `pin_utils` to pin to the stack safely, just remember that they use
    // unsafe under the hood so it's like using an already-reviewed unsafe
    // implementation.

    let mut pinned1 = Box::pin(gen1);
    let mut pinned2 = Box::pin(gen2);

    // Uncomment these if you think it's safe to pin the values to the stack instead 
    // (it is in this case). Remember to comment out the two previous lines first.
    //let mut pinned1 = unsafe { Pin::new_unchecked(&mut gen1) };
    //let mut pinned2 = unsafe { Pin::new_unchecked(&mut gen2) };

    if let GeneratorState::Yielded(n) = pinned1.as_mut().resume() {
        println!("Gen1 got value {}", n);
    }
    
    if let GeneratorState::Yielded(n) = pinned2.as_mut().resume() {
        println!("Gen2 got value {}", n);
    };

    // This won't work:
    // std::mem::swap(&mut gen, &mut gen2);
    // This will work but will just swap the pointers so nothing bad happens here:
    // std::mem::swap(&mut pinned1, &mut pinned2);

    let _ = pinned1.as_mut().resume();
    let _ = pinned2.as_mut().resume();
}

enum GeneratorState<Y, R> {
    Yielded(Y),  
    Complete(R), 
}

trait Generator {
    type Yield;
    type Return;
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return>;
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

// This tells us that the underlying pointer is not safe to move after pinning.
// In this case, only we as implementors "feel" this, however, if someone is
// relying on our Pinned pointer this will prevent them from moving it. You need
// to enable the feature flag ` #![feature(optin_builtin_traits)]` and use the
// nightly compiler to implement `!Unpin`. Normally, you would use
// `std::marker::PhantomPinned` to indicate that the struct is `!Unpin`.
impl !Unpin for GeneratorA { }

impl Generator for GeneratorA {
    type Yield = usize;
    type Return = ();
    fn resume(self: Pin<&mut Self>) -> GeneratorState<Self::Yield, Self::Return> {
        // lets us get ownership over current state
        let this = unsafe { self.get_unchecked_mut() };
            match this {
            GeneratorA::Enter => {
                let to_borrow = String::from("Hello");
                let borrowed = &to_borrow;
                let res = borrowed.len();
                *this = GeneratorA::Yield1 {to_borrow, borrowed: std::ptr::null()};

                // Trick to actually get a self reference. We can't reference
                // the `String` earlier since these references will point to the
                // location in this stack frame which will not be valid anymore
                // when this function returns.
                if let GeneratorA::Yield1 {to_borrow, borrowed} = this {
                    *borrowed = to_borrow;
                }

                GeneratorState::Yielded(res)
            }

            GeneratorA::Yield1 {borrowed, ..} => {
                let borrowed: &String = unsafe {&**borrowed};
                println!("{} world", borrowed);
                *this = GeneratorA::Exit;
                GeneratorState::Complete(())
            }
            GeneratorA::Exit => panic!("Can't advance an exited generator!"),
        }
    }
}
```

现在，正如你所看到的，这个 API 的使用者必须:
1. 将值装箱，从而在堆上分配它
2. 使用`unafe`然后把值固定到栈上。 用户知道如果他们事后移动了这个值，那么他们在就违反了当他们使用unsafe时候做出的承诺,也就是一直持有.

希望在这之后，你会知道当你在一个异步函数中使用`yield`或者`await`关键词时会发生什么，以及如果我们想要安全地跨yield/await借用时。,为什么我们需要`Pin`
