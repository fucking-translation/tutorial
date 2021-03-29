# Hello Tokio

[原文](https://tokio.rs/)  

</br>

我们从写一个最基础的的`Tokio`程序开始，这个程序会连接到`MiniRedis`的服务端，然后设置一个`key`为`hello`，`value`为`world`的键值对，然后再把这个键值对读取回来。这些操作我们会使用名为`Mini-Redis`的客户端库来完成。

## The Code

### 创建一个新程序

我们从创建一个新的`Rust`程序开始

```s
cargo new my-redis
cd my-redis
```

### 添加依赖

接下来，打开`Cargo.toml`，并在`[dependencies]`后添加下面的代码

```toml
tokio = { version = "1", features = ["full"] }
mini-redis = "0.4"
```

### 开始写代码

然后，打开`main.rs`并用下面的代码替换文件的内容

```rust
use mini_redis::{client, Result};

#[tokil::main]
pub async fn main() -> Result<()> {
  // Open a connection to the mini-redis address.
  let mut client = client::connect("127.0.0.1:6379").await?;
  
  // Set the key "hello" with value "world"
  client.set("hello", "world".into()).await?;
  
  // Get key "hello"
  let result = client.get("hello").await?;
  
  println!("got value from the server; result={:?}", result);
  
  Ok(())
}
```

为了确保`Mini-Redis`的服务端处于运行状态，我们打开一个终端窗口，运行如下命令:

```s
mini-redis-server
```

接下来运行我们的`mini-redis`程序

```s
$ cargo run
got value from the server; result=Some(b"world)
```

成功了！

> 你可以从[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/hello-tokio/src/main.rs)找到完整的源码.

## Break it down

接下来花点时间梳理下我们刚才做的事情。代码并不多，但其中却触发了许多的事情。

```rust
let mut client = client::connect("127.0.0.1:6379").await?;
```

函数`client::connect`是`mini-redis`这个包所提供的，他会使用指定的地址来异步的创建一个`TCP`连接，当这个连接建立成功时，`client`则保存了该函数返回的结果。尽管这个操作是异步发生的，但代码 看起来 却是同步的。其中唯一指示了该操作为异步的只有`.await`操作符。

### 什么是异步编程?

大部分的电脑程序都按照他们代码所写的顺序执行，最前面的先执行，然后是下一行，然后一直执行下去。在同步编程中，当程序遇到了一个无法立即完成的操作时，他会堵塞在该位置一直到操作完成，举个例子，在创建`TCP`连接时连接双方需要在网络中交换一些信息，交换信息的操作需要花费相当的时间，而运行这段代码的线程在这个时间内将被阻塞。

在异步编程中，如果一个操作不能马上完成的话，他将被暂停然后切换到后台等待，执行的线程不会被阻塞，因此他可以继续执行其他的事情。当这个操作完成时，他又会被切换至前台并从之前中断的地方继续执行。我们刚刚实现的示例只启动了一个任务，所以在这个任务的操作被暂停时并没有发生任何其他的事情，但通常异步的编程会同时运行许多的任务。

尽管异步编程能够给我们带来更快的程序，与此同时他也为程序带来了更高的复杂度。开发人员为了能够在异步操作完成时将任务重新恢复执行，需要去跟进任务的运行状态。从历史经验上来看，这是一个乏味并且非常容易出错的工作。

### 编译时的绿色线程

**Rust**使用了`async/await`特性来实现了异步编程的功能。会执行异步操作的函数通过`async`关键字进行标识，在我们的示例中，`connect`函数进行了如下的定义：

```rust
use mini_redis::Result;
use mini_redis::client:Client;
use tokio::net::ToSocketAddrs;

pub async fn connect<T: ToSocketAddrs>(addr: T) -> Result<Client> {
  // ...
}
```

`async fn`的定义跟同步函数很类似，但他以异步的方式执行。`Rust`在编译时将代码转换为异步的操作，所有使用`.await`调用并定义为`async fn`的操作将让出线程的执行权。这样该线程就能在异步操作被放到后台的期间做其他的事情。

尽管也有一些其他的编程语言实现`async/await`的特性，但`Rust`使用了一个独立的方式，最主要的一点是`Rust`的异步操作是`lazy`的。这导致了运行时的语义跟其他编程语言的产生了区别。

如果到现在还没弄得很明白，不用担心，我们还会继续在接下来的篇幅中探讨`async/await`。

### 使用 async/await

异步函数能够与普通的`Rust`函数一样使用。但是，调用这些函数不意味着执行这些函数，调用`async fn`类型的函数返回的是一个代表该操作的标识。在概念上他跟一个无参的闭包函数类型。为了能够真正的执行它，你需要在函数返回的标识上使用`.await`操作。

我们来看看下面的例子

```rust
async fn say_world() {
  println!("world");
}

#[tokio::main]
async fn main() {
  // Calling `say_world()` does not execute the body of `say_world()`
  let op = say_hello();
  
  // This println! comes first
  println!("hello");
  
  // Calling `.await` on `op` starts executing `say_world`.
  op.await;
}
```

输出

```s
hello
world
```

`async fn`函数的返回结果是一个实现了`Future`trait 的匿名类型。

所以这里到底是怎么执行的，还得看`Rust`最终转换出的代码及`Future`的定义，后续我会单独细讲

### 异步的`main`函数

用来启动程序的`main`函数其他普通的`Rust`程序的有所不同：

1. 被定义为`async fn`
2. 添加了`#[tokio::main]`宏

`async fn`函数在我们需要执行异步操作的上下文中被使用。然而，异步函数需要通过`runtime`来运行，`runtime`中包含异步任务的调度器，他提供了事件驱动的**I/O**、定时器等。`runtime`并不会自动的运行，所以需要在主函数中运行它。

我们在`async fn main()`函数中添加的`#[tokio::main]`宏会将其转换为同步的`fn main()`函数，该函数会初始化 `runtime`并执行我们定义的异步的`main`函数。

比如

```rust
#[tokio::main]
async fn main() {
  println!("hello");
}
```

会被转换为

```rust
fn main() {
  let mut rt = tokio::runtime::Runtime::new().unwrap();
  rt.block_on(async {
    println!("hello");
  })
}
```

Tokio 中具体的`runtime`的细节在后续的章节中会补充。

### Cargo 特性

我们在定义对 Tokio 的依赖时使用`full`特性。

```toml
tokio = { version = "1", features = ["full"] }
```

Tokio 提供了大量的功能 (TCP, UDP, Unix sockets, Timers, sync utilities, multiple scheduler types 等)，但并不是所有的程序都需要用到这么多的功能。在需要缩短编译时间或减小程序大小时，可以只选择所需的特性。

现在的话，还是继续使用`full`特性吧。