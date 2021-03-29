# Spawning

[原文](https://tokio.rs/)

</br>

我们要开始换挡加速开始学习`Redis`服务端了。

首先，将我们上一节写的`Set/Get`代码移到示例目录`examples`，这样我们可以让他跟服务端代码一起运行。

```console
mkdir -p examples
mv src/main.rs examples/hello-redis.rs
```

接下来创建一个新的`src/main.rs`然后继续。

## Accepting Sockets

我们的`Redis`服务端第一步需要做的是接收一个`TCP`套接字，这个操作通过 `tokio::net::TcpListener`完成。

`Tokio`大部分的的类型名称都定义成跟`Rust`标准库中同步类型一样。在有必要的情况下，`Tokio`会以`async fn`的形式，提供与标准库一样的 API。

`TcpListener`绑定到了`6379`端口，套接字则会在循环中被接收，每个套接字都会在处理完之后关闭。就目前而言，我们会从中读取命令打印到标准输出，然后返回一个错误。

```rust
use mini_redis::{Connection, Frame};
use tokio::net::{TcpListener, TcpStream};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    loop {
        // The second item contains the IP and Port or the new connection
        let (socket, _) = listener.accept().await.unwrap();
        process(socket).await;
    }
}

async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frame** instead of
    // byte streams. The `Connection` type is defined by mini-redis
    let mut connection = Connection::new(socket);
    if let Some(frame) = connection.read_frame().await.unwrap() {
        println!("GOT: {:?}", frame);

        // Response with an error
        let response = Frame::Error("unimplemented".to_string());
        connection.write_frame(&response);
    }
}
```

接下来，启动这个接收循环：

```s
$ cargo run
```

接下来在一个独立的终端窗口，启动`hello-redis`示例 (上一节实现的`SET/GET`)

```s
$ cargo run --example hello-redis
```

输出为：

```s
Error: "unimplemented"
```

在服务端的终端输出如下：

```s
GOT: Array([Bulk(b'set'), Bulk(b'hello'), Bulk(b'world')])
```

## Concurrency

我们的服务端还有一个小问题*(除了返回错误)*，他每次只能处理一个请求。当接收了一个连接后，服务端会在当前循环中一直堵塞到完全把返回信息写到套接字中。

我们希望`Redis`服务能够同时处理多个请求，所以我们需要让他并发 (`Concurrenty`) 起来。

并发跟并行并不是同一种概念。如果你能够交替着执行两个任务，那你这两个任务可以说是并发但不是并行的。为了能够让他并发起来，你需要两个人，每个人各自处理一个任务。

使用`Tokio`的一个好处就是异步的代码让你能够在不使用多线程的前提下让多个任务并发执行。事实上，`Tokio`能够在单线程中并发运行非常多的任务！

为了能够并发的处理连接，我们需要为每个到达的连接创建一个新的任务，然后让这个任务负责处理该连接。

接收连接的循环现在变成了这样：

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() {
  let listener = TcpListener::bind("127.0.0.1:6379")
  
  loop {
    let (socket, _) = listerner.accept().await.unwrap();
    // A new task is spqwned for each inbound socket. the socket is
    // moved to the new task and processed there.
    tokio::spawn(async move {
      process(socket).await;
    });
  }
}
```

### Tasks

`Tokio`的任务是异步的绿色线程，他通过传递给`tokio::spawn`的`async`语句块创建，这个函数接收`async`语句块后返回一个`JoinHandle`，调用者则通过 `JoinHandle`与创建的任务交互。有些传递的`async`语句块是具有返回值的，调用者通过`JoinHandle`的`.await`来获取其返回值，

```rust
#[tokio::main]
async fn main() {
  let handle = tokio::spawn(async {
    "return value"
  });
  
  // Do some other work
  
  let out = handle.await.unwrap();
  println!("GOT {}", out);
}
```

在`JoinHandle`上执行`.await`等待会得到一个`Result`。当任务在执行时遇到了错误时，`JoinHandle`会返回`Err`，这会在任务发生错误，或是因为`Runtime`被强制关闭而导致任务被强制取消时产生。

任务在`Tokio`中是非常轻量的，实际上他只会需要申请一次`64`个字节的内存。所以程序可以轻松的产生成千上万的任务。

### `'static` bound

当你通过`Tokio`的`Runtime`创建一个任务时，这个任务的类型必须是`'static`的。这意味着被创建的任务不能够包含对任务以外任何数据的引用。

有一个常见的误解是`'static`始终代表着一直存活。但在这个场景中并不是，标识为`'static`的值只是意味着他不会产生内存泄漏。具体的可以通过[Common Rust Lisetime Misconceptions](https://github.com/pretzelhammer/rust-blog/blob/master/posts/common-rust-lifetime-misconceptions.md#2-if-t-static-then-t-must-be-valid-for-the-entire-program)进行了解。

举个例子，下面的代码无法通过编译：

```rust
use tokio::task;

#[tokio::main]
async fn main() {
  let v = vec![1, 2, 3];
  
  task.spawn(async {
    println!("Here's a vec: {:?}", v);
  });
}
```

尝试编译的话，会得到下面的错误信息：

```s
error[E0373]: async block may outlive the current function, but
              it borrows `v`, which is owned by the current function
 --> src/main.rs:7:23
  |
7 |       task::spawn(async {
  |  _______________________^
8 | |         println!("Here's a vec: {:?}", v);
  | |                                        - `v` is borrowed here
9 | |     });
  | |_____^ may outlive borrowed value `v`
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:7:17
  |
7 |       task::spawn(async {
  |  _________________^
8 | |         println!("Here's a vector: {:?}", v);
9 | |     });
  | |_____^
help: to force the async block to take ownership of `v` (and any other
      referenced variables), use the `move` keyword
  |
7 |     task::spawn(async move {
8 |         println!("Here's a vec: {:?}", v);
9 |     });
  |
```

这些错误信息源自于在默认条件下，变量并不会`moved`到异步的代码块中，`v`向量在这个时候仍由`main`函数拥有，但在`println!`这一行产生了对`v`的借用。`Rust`的编译器的错误信息还提供了修复这个错误的建议：将第 7 行改为`task::spawn(async move {`能够指示编译器将变量`v`移动到创建的任务中，这样的话该任务就会拥有所有他所依赖的数据，让自己满足`'static'`。

如果一个数据需要同时被多个任务并发的访问，那他应该使用同步机制来进行共享，比如使用`Arc`。

同时我们也留意到错误信息提到参数类型的存活时间超过了`'static`，这个术语可能会造成困惑，因为`'static`的生命周期已经覆盖了整个程序了，如果参数的存活时间还超过了`'static`是不是意味着存在内存泄漏？对于这个疑惑的具体解释是，这里提到的超过`'static`生命周期的是参数的类型而不是参数的值，当参数的值被不再使用时他就会被销毁了。

当我们提到一个值是`'static`时，说他会永远存活并不意味着不正确。这点非常重要，因为编译器无法确定这个新创建的任务会存活多久，所以他能做的确保这个任务不会存活太久的方式就是让他一直都存在。

上面提供的链接中提到的术语 "bounded by 'static" 或 "its type outlives 'static" 或 "the value is 'static for T: 'static" 都表达了同一个意思，但他们跟以`&'static T`使用的标注是不一样的。

### `Send` bound

通过`tokio::spawn`创建的任务必须实现了`Send`语义，这样`Tokio`的`Runtime`才能在他们因为执行`.await`被暂停时将他们切换到不同的线程中。

任务在他调用`.await`时拥有的所有数据都是`Send`时满足`Send`的条件，这听起来有点微妙。当`.await`被调用时，任务会让出执行权给调度器，在他下一次被执行时则从上一次让出的位置开始。为了能实现这个机制，所有的状态信息都会在执行的线程间进行转移，这样任务才能在执行线程间转移。相反，如果他的状态不满足`Send`，那他就不满足作为一个任务的条件。

举个正常运行的例子

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
  tokio::spawn(async {
    {
      let rc = Rc::new("hello");
      println!("{}", rc);
    }
    
    // `rc` is no longer used. It is **not** persisted when
    // the task yields to the scheduler
    yield_now().await;
  });
}
```

下面的例子则无法编译：

```rust
use tokio::task::yield_now;
use std::rc::Rc;

#[tokio::main]
async fn main() {
  tokio::spawn(async {
    let rc = Rc::new("hello");
    
    // `rc` is used after `.await`. It must be persisted to
    // the task's state.
    yield_now().await;
    
    println!("{}", rc);
  });
}
```

尝试编译时，会产生下面的错误信息：

```s
error: future cannot be sent between threads safely
   --> src/main.rs:6:5
    |
6   |     tokio::spawn(async {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    | 
   ::: [..]spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in
    |                          `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait
    |       `std::marker::Send` is not  implemented for
    |       `std::rc::Rc<&str>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:10:9
    |
7   |         let rc = Rc::new("hello");
    |             -- has type `std::rc::Rc<&str>` which is not `Send`
...
10  |         yield_now().await;
    |         ^^^^^^^^^^^^^^^^^ await occurs here, with `rc` maybe
    |                           used later
11  |         println!("{}", rc);
12  |     });
    |     - `rc` is later dropped here
```

我们会在[下一章](./sharedstate.md)讨论一个关于这个错误的特殊的例子。

### Store Values

接下来继续实现`process`函数来处理接收的命令。我们将使用 `HashMap`来存储收到的值，`SET`操作会插入一条新的记录到`HashMap`中，而`GET`操作则从中读取。并且，我们还会使用一个循环来处理来自同个连接的多个命令。

```rust
async fn process(socket: TcpStream) {
    // The `Connection` lets us read/write redis **frame** instead of
    // byte streams. The `Connection` type is defined by mini-redis

    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // A hashmap is used to store data
    let mut db = HashMap::new();

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                db.insert(cmd.key().to_string(), cmd.value().to_vec());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                if let Some(value) = db.get(cmd.key()) {
                    Frame::Bulk(value.clone().into())
                } else {
                    Frame::Null
                }
            }
            cmd => panic!("unimplemented {:?}", cmd),
        };

        connection.write_frame(&response).await.unwrap();
    }
```

接下来启动服务端：

```s
$ cargo run
```

然后打开一个新的终端窗口，运行`hello-redis`示例：

```rust
$ cargo run --example hello-redis
```

然后，我们就能看到下面的输出了：

```s
got value from the server; success=Some(b'world')
```

现在我们能获取跟设置信息了，但还存在一个问题。设置的信息还没办法在不同的连接中共享，如果其他的套接字连接尝试使用`GET`命令获取 `hello`的值，他将找不到任何东西。

完整的代码在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/spawning/src/main.rs)。

在下一节，我们会为所有的客户端实现一个共享、持久化的存储。