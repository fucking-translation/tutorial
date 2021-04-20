# 理解 Rust 中的 futures - 第一节

[原文](https://www.viget.com/articles/understanding-futures-in-rust-part-1/)

</br>

> 2019年8月2日：这篇文章已经更新。他最初是为了匹配 future-rs 库的 0.1 版本而写，但是 futures 已经在标准库中趋于稳定并有了一些显著的改变。这篇文章将会涵盖许多与之前相同的材料，而且还探讨了使用`std::task`模块创建一个简单的执行器。  
> 2019年8月15日：第二节已经发表！请查看[这里](./part2.md)

## 背景

Rust 中的 futures 类似于 (are analogous to) JavaScript 中的 [promises](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise)。它们是 Rust 中并发原语 (concurrency primitives) 的强大抽象。它们也是 [async/await](https://areweasyncyet.rs/) 的一块垫脚石，其中 [async/await](https://areweasyncyet.rs/) 可以让我们像写同步代码一样编写异步代码。

[async/await](https://areweasyncyet.rs/) 尚未在 Rust 中做好准备，但是这不是你在 Rust 项目中不使用 futures 的理由。[tokio](https://tokio.rs/) 是稳定的，易于使用且快如闪电。请查阅[这篇文章](https://tokio.rs/docs/futures/overview/)，以获取有关使用 futures.* 的入门知识。

futures 已经存在于标准库中，但是在本系列文章中，我将编写一个 futures 库的简单版本，来演示它的基本原理，如何使用，以及要规避哪些陷阱 (pitfalls)。

tokio 在 master 分支使用 std::future，但所有的文档都涉及 0.1 版本的 futures。但是，这些概念都是相通的。

尽管 futures 已经在标准库中，但是却缺少许多常用的特性。它们现在位于 [futures-preview](https://docs.rs/futures-preview/0.3.0-alpha.17/futures/) 中，我将会引用一些定义在这里的函数和特征。事物是飞速发展的，这个库中的许多内容最终都将合到标准库中。

## 先决条件

- 少量的 Rust 知识或者有自驱学习的能力 (阅读 [rust 程序设计](https://github.com/KaiserY/trpl-zh-cn)，这本书写的很棒)。
- 一个现代浏览器，如 Chrome，Firefox，Safari 或者 Edge (我们将需要使用 [rust playground](https://play.rust-lang.org/))。

## 目标

这篇文章的目标是理解下面这段代码，并实现一些类型以及函数让这段代码能够编译。这对于标准库中的 futures 来说是一段合法的语法，并演示了 futures 链的工作原理。

```rust
// This does not compile, yet

fn main() {
    let future1 = future::ok::<u32, u32>(1)
        .map(|x| x + 3)
        .map_err(|e| println!("Error: {:?}", e))
        .and_then(|x| Ok(x - 3))
        .then(|res| {
          match res {
              Ok(val) => Ok(val + 3),
              err => err,
          }
        });
    let joined_future = future::join(future1, future::err::<u32, u32>(2));
    let val = block_on(joined_future);
    assert_eq!(val, (Ok(4), Err(2)));
}
```

## future 究竟是什么？

具体来说，它是一系列异步计算所代表的的值。futures 库的文档将其称为：“它是一个尚未准备好的值的代理对象”。

Rust 中的 futures 允许你定义一个任务，如将要异步执行的网络调用或者计算。你可以在其结果之上构建一个调用链，将其转换，错误处理以及与其他 futures 合并，并在其上进行许多其他的计算。它们将只当 futures 被传递给一个执行器(如 tokio 的`run`函数)时才会被执行。事实上，如果你在 futures 超出范围之前都不使用它，什么事都不会发生。因此，futures 库声明了 futures `must_use`，并当你允许它们即使超出范围也可以不对其进行消耗时，编译器将会提供一个警告。

如果你对 JavaScript 的 promises 很熟悉，这看起来可能会很奇怪。在 JavaScript 中，promises 会在事件循环中执行并且没有其他选择。“执行器”函数会立即执行。但是本质上，promises 依然简单的定义了一套将要运行的指令。在 Rust 中，执行器有很多种运行的策略。

## 构建我们的 future

在上层视角，我们需要一些组件来使 futures 正常工作：一个执行器，一个 future 特征和一个 poll 类型。

### 执行器

如果我们没有办法来执行一个 future，它将不会做任何事。因为我们实现的是我们自己的 futures，因此我们也需要实现一个执行器。在这个练习中，我们实际上不会做任何异步操作，但是我们类似于异步调用。futures 是基于 pull 而不是基于 push。这让它们成为零成本抽象，也意味着它们将被轮询一次，并在准备再次轮询时负责通知执行器。它工作的细节对理解 futures 是如何创建以及组成调用链不是很重要，因此我们的执行器十分简单粗暴。它只能执行一个 future，并且它不能做任何有意义的异步操作。tokio 的文档对 futures 的运行时模型做了更多的介绍。

这里有一个非常简单的实现：

```rust
use std::cell::RefCell;

thread_local!(static NOTIFY: RefCell<bool> = RefCell::new(true));

struct Context<'a> {
    waker: &'a Waker,
}

impl<'a> Context<'a> {
    fn from_waker(waker: &'a Waker) -> Self {
        Context { waker }
    }

    fn waker(&self) -> &'a Waker {
        &self.waker
    }
}

struct Waker;

impl Waker {
    fn wake(&self) {
        NOTIFY.with(|f| *f.borrow_mut() = true)
    }
}

fn run<F>(mut f: F) -> F::Output
where
    F: Future,
{
    NOTIFY.with(|n| loop {
        if *n.borrow() {
            *n.borrow_mut() = false;
            let ctx = Context::from_waker(&Waker);
            if let Poll::Ready(val) = f.poll(&ctx) {
                return val;
            }
        }
    })
}
```

`run`是 F 类型的泛型函数，其中 F 一个 future，它返回一个定义在`Future`特征中的`Output`类型的值。我们将在稍后对其进行介绍。

函数的主体近似于一个真实执行器想要做得事情，它一直循环，直到接收到 futures 的通知：future 已经准备好再次被轮询。当 future 就绪时它在函数进行返回。`Context`和`Waker`类型是对定义在[这里](https://docs.rs/futures-preview/0.3.0-alpha.17/futures/task/index.html)的`future::task`模块中同名类型的模拟。它们需要在此处进行编译，但这超出了本文的范围。你可以自行深入了解它们的工作原理。

Poll 是一个简单的泛型枚举，其定义如下所示：

```rust
enum Poll<T> {
    Ready(T),
    Pending
}
```

### 我们的 Trait

[Trait](https://doc.rust-lang.org/book/ch10-02-traits.html) 是 Rust 中定义共享行为的一种方式。它可以让我们指定实现类型时必须定义的类型和函数。它也可以实现默认的行为，我们将会在组合器中看到。

我们的实现的特征如下所示(它实际与真实 futures 中的定义相同)：

```rust
trait Future {
    type Output;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output>;
}
```

这个特征看起来很简单，并简单的声明了需要的`Output`类型以及唯一需要实现的引用了 context 对象的`poll`函数。这个对象有一个 waker 的引用，它被用来通知运行时 future 已经准备好再一次被轮询了。

### 我们的实现

```rust
#[derive(Default)]
struct MyFuture {
    count: u32,
}

impl Future for MyFuture {
    type Output = i32;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output> {
        match self.count {
            3 => Poll::Ready(3),
            _ => {
                self.count += 1;
                ctx.waker().wake();
                Poll::Pending
            }
        }
    }
}
```

让我们逐行进行说明：

- `#[derive(Default)]`自动为类型创建了一个`::default()`函数。数值类型默认被设置为 0。
- `struct MyFuture { count: u32}` 定义了一个简单的带有计数器的结构体。它可以用来模拟异步操作。
- `impl Future for MyFuture`是我们对应的 Future 实现。
- 我们将 Output 设置为`i32`因此我们可以返回内部的计数值。
- 在我们的`poll`实现中我们决定了根据内部的计数值可以做些什么。
- 如果它匹配到了 3，`3 =>`将会返回一个带有 3 的`Poll::Ready`响应
- 如果是其他情况我们将增加计数器的值并返回`Poll::Pending`

在十分简单的主函数中，运行我们的 future！

```rust
fn main() {
    let my_future = MyFuture::default();
    println!("Output: {}", run(my_future));
}
```

[在 rust playground 中运行](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=254b419cb4a9229b67219400890c9e9b)

### 最后一步

它正常的运行，但是并没有向你展示 futures 的威力。因此让我们创建一个超级方便的 future 以将其链接起来，让它在可以添加 1 的类型中加 1。举个例子：

```rust
struct AddOneFuture<T>(T);

impl<T> Future for AddOneFuture<T>
where
    T: Future,
    T::Output: std::ops::Add<i32, Output = i32>,
{
    type Output = i32;

    fn poll(&mut self, ctx: &Context) -> Poll<Self::Output> {
        match self.0.poll(ctx) {
            Poll::Ready(count) => Poll::Ready(count + 1),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

它看起来很复杂但实际上很简单。我将再一次逐行对其进行讲解：

- `struct AddOneFuture<T>(T);`这是一个 newtype 泛型模式的示例。它可以让我们封装其他的结构体并在其中添加我们的行为。
- `impl<T> Future for AddOneFuture<T>`是一个特征的泛型实现。
- `T: Future`确保任何被封装在 AddOneFuture 中的结构都实现了 Future。
- `T::Item: std::ops::Add<i32, Output = i32>`确保由`Poll::Ready(value)`代表的值可以响应`+`操作。

剩下的可以完美的自解释。它使用传入了 context 的`self.0.poll`轮询了内部的 future，并根据结构返回`Poll::Pending`或返回内部 future 的加 1 计数，即`Poll::Ready(count + 1)`。

我们进更新了主函数以使用新的 future。

```rust
fn main() {
    let my_future = MyFuture::default();
    println!("Output: {}", run(AddOneFuture(my_future)));
}
```

[在 rust playground 中运行](https://play.rust-lang.org/?version=stable&mode=debug&edition=2018&gist=82df3f3ae9ab242d1d536bf9c851349d)

现在，我们开始探索如何使用 future 将异步操作链接在一起。建立这些链接功能(组合器)只需几步简单的步骤，这些功能可以给 future 带来强大的威力。

### 回顾 (Recap)

- Futures 是一种强大的方式，可以利用 Rust 的零成本抽象来编写可读，快速的异步代码。
- Futures 就像是 JavaScript 或其他语言中的 promises。
- 我们学习了如何构建泛型类型以及一些关于链式操作的知识。

## 接下来

在[第二节](./part2.md)中，我们将介绍组合器(非技术意义上)，它可以让你使用函数(如回调)来构建新的类型。如果你使用过 JavaScript 中的 promises，这对你来说将会十分熟悉。