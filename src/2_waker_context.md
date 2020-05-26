##  唤醒器和上下文(Waker and Context)

### 概述

1. 了解 Waker 对象是如何构造的
2. 了解运行时如何知道`leaf-future`何时可以恢复
3. 了解动态分发的基础知识和trait对象

`Waker`类型在[RFC#2592](https://github.com/rust-lang/rfcs/blob/master/text/2592-futures.md#waking-up)中介绍.


### 唤醒器

`Waker`类型允许在运行时的reactor 部分和执行器部分之间进行松散耦合。

通过使用不与`Future`执行绑定的唤醒机制，运行时实现者可以提出有趣的新唤醒机制。 例如，可以生成一个线程来执行一些工作，这些工作结束时通知`Future`，这完全独立于当前的运行时。

如果没有唤醒程序，执行程序将是通知正在运行的任务的唯一方式，而使用唤醒程序，我们将得到一个松散耦合，其中很容易使用新的`leaf-future`来扩展生态系统。

> 如果你想了解更多关于 Waker 类型背后的原因，我可以推荐[Withoutboats articles series about them](https://boats.gitlab.io/blog/post/wakers-i/)。

### 理解唤醒器

在实现我们自己的`Future`时，我们遇到的最令人困惑的事情之一就是我们如何实现一个唤醒器。 创建一个 Waker 需要创建一个 vtable，这个vtable允许我们使用动态方式调用我们真实的Waker实现.

> 如果你想知道更多关于Rust中的动态分发，我可以推荐 Adam Schwalm 写的一篇文章 [Exploring Dynamic Dispatch in Rust](https://alschwalm.com/blog/static/2017/03/07/exploring-dynamic-dispatch-in-rust/).

让我们更详细地解释一下。

### Rust中的胖指针

为了更好地理解我们如何在 Rust 中实现 Waker，我们需要退后一步并讨论一些基本原理。 让我们首先看看 Rust 中一些不同指针类型的大小。

运行以下代码:
```rust
trait SomeTrait { }

fn main() {
    println!("======== The size of different pointers in Rust: ========");
    println!("&dyn Trait:-----{}", size_of::<&dyn SomeTrait>());
    println!("&[&dyn Trait]:--{}", size_of::<&[&dyn SomeTrait]>());
    println!("Box<Trait>:-----{}", size_of::<Box<SomeTrait>>());
    println!("&i32:-----------{}", size_of::<&i32>());
    println!("&[i32]:---------{}", size_of::<&[i32]>());
    println!("Box<i32>:-------{}", size_of::<Box<i32>>());
    println!("&Box<i32>:------{}", size_of::<&Box<i32>>());
    println!("[&dyn Trait;4]:-{}", size_of::<[&dyn SomeTrait; 4]>());
    println!("[i32;4]:--------{}", size_of::<[i32; 4]>());
}
```
从运行后的输出中可以看到，引用的大小是不同的。 许多是8字节(在64位系统中是指针大小) ，但有些是16字节。

16字节大小的指针被称为“胖指针” ，因为它们携带额外的信息。


例如 `&[i32]`:
- 前8个字节是指向数组中第一个元素的实际指针(或 slice 引用的数组的一部分)
- 第二个8字节是切片的长度


例如  `&dyn SomeTrait`:
 
这就是我们将要关注的胖指针的类型。`&dyn SomeTrait` 是一个trait的引用，或者 Rust称之为一个trait对象。

指向 trait 对象的指针布局如下:
- 前8个字节指向trait 对象的data
- 后八个字节指向trait对象的 vtable

这样做的好处是，我们可以引用一个对象，除了它实现了 trait 定义的方法之外，我们对这个对象一无所知。 为了达到这个目的，我们使用动态分发。

让我们用代码而不是文字来解释这一点，通过这些部分来实现我们自己的 trait 对象:

```rust
// A reference to a trait object is a fat pointer: (data_ptr, vtable_ptr)
trait Test {
    fn add(&self) -> i32;
    fn sub(&self) -> i32;
    fn mul(&self) -> i32;
}

// This will represent our home brewn fat pointer to a trait object
   #[repr(C)]
struct FatPointer<'a> {
    /// A reference is a pointer to an instantiated `Data` instance
    data: &'a mut Data,
    /// Since we need to pass in literal values like length and alignment it's
    /// easiest for us to convert pointers to usize-integers instead of the other way around.
    vtable: *const usize,
}

// This is the data in our trait object. It's just two numbers we want to operate on.
struct Data {
    a: i32,
    b: i32,
}

// ====== function definitions ======
fn add(s: &Data) -> i32 {
    s.a + s.b
}
fn sub(s: &Data) -> i32 {
    s.a - s.b
}
fn mul(s: &Data) -> i32 {
    s.a * s.b
}

fn main() {
    let mut data = Data {a: 3, b: 2};
    // vtable is like special purpose array of pointer-length types with a fixed
    // format where the three first values has a special meaning like the
    // length of the array is encoded in the array itself as the second value.
    let vtable = vec![
        0,            // pointer to `Drop` (which we're not implementing here)
        6,            // lenght of vtable
        8,            // alignment

        // we need to make sure we add these in the same order as defined in the Trait.
        add as usize, // function pointer - try changing the order of `add`
        sub as usize, // function pointer - and `sub` to see what happens
        mul as usize, // function pointer
    ];

    let fat_pointer = FatPointer { data: &mut data, vtable: vtable.as_ptr()};
    let test = unsafe { std::mem::transmute::<FatPointer, &dyn Test>(fat_pointer) };

    // And voalá, it's now a trait object we can call methods on
    println!("Add: 3 + 2 = {}", test.add());
    println!("Sub: 3 - 2 = {}", test.sub());
    println!("Mul: 3 * 2 = {}", test.mul());
}
```

稍后，当我们实现我们自己的 Waker 时，我们实际上会像这里一样建立一个 vtable。 我们创造它的方式略有不同，但是现在你知道了规则特征对象是如何工作的，你可能会认识到我们在做什么，这使得它不那么神秘。

### 奖励部分

您可能想知道为什么Waker是这样实现的，而不仅仅是作为一个普通的trait.

原因在于灵活性。 以这里的方式实现 Waker，可以很灵活地选择要使用的内存管理方案。

“正常”的方法是使用 Arc 来使用引用计数来跟踪 Waker 对象何时可以被删除。 但是，这不是唯一的方法，您还可以使用纯粹的全局函数和状态，或者任何其他您希望的方法。

这在表中为运行时实现者留下了许多选项。

