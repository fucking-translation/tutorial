# Select

[原文](https://tokio.rs/)

</br>

到目前为止，在需要可以并发运行程序时，可以通过 `spawn` 创建一个新的任务，现在我们来学一下 Tokio 的一些其他执行异步代码的方式。

## tokio::select!

`tokio::select!` 宏允许我们等待多个异步的任务，并且在其中一个完成时返回，比如

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
  let (tx1, rx1) = oneshot::channel();
  let (tx2, rx2) = oneshot::channel();
  
  tokio::spawn(async {
    let _ = tx1.send("one");
  });
  
  tokio::spawn(async {
    let _ = tx2.send("two");
  });
  
  tokio::select! {
    val = rx1 => {
      println!("rx1 completed first with {:?}", val);
    }
    val = rx2 => {
      println!("rx2 completed first with {:?}", val);
    }
  }
}
```

这里我们使用了两个 OneShot Channel, 每个 Channel 都可能会先完成，`select!` 语句同时等待这两个 Channel，并在操作完成时将其返回值绑定到语句块的 `val` 变量，然后执行对应的完成语句。

要注意的是，另一个未完成的操作将会被丢弃，在这个示例中，对应的操作是等待每一个 `oneshot::Receiver` 的结果，最后未完成的那个 `Channel` 将会被丢弃。

## Cancellation

在 Rust 的异步系统中，取消操作通过丢弃对应的 `Future` 实现，回顾 `Async in depth` 一节，Rust 的异步操作是通过惰性执行的 `Future` 来实现的，这些操作只会在对 `Future` 进行 Poll 的时候执行，如果 `Future` 被丢弃，则其对应的操作将无法继续推进，因为操作对应的状态已经丢失了。

有时候异步的操作会用来创建后台任务或者其他运行在后台的操作，比如在上面的示例中，一个任务被创建来发送信息，通常来说这些任务会通过计算来产生一些数据。

`Future` 或其他的类型可以通过实现 `Drop` 来实现资源清理的操作，Tokio 的 `oneshot::Receiver` 通过实现 `Drop` 来给 `Sender` 发送关闭的通知，`Sender` 会在收到通知的时候可以通过 通过 `Drop` 其他的资源来清理进程内的信息。

```rust
use tokio::sync::oneshot;

async fn some_operation() -> String {
  // Compute value here
}

#[tokio::main]
async fn main() {
  let (mut tx1, rx1) = oneshot::channel();
  let (tx2, rx2) = oneshot::channel();
  
  tokio::spawn(async {
    // Selection on the operation and the oneshot's
    // `closed()` notification
    tokio::select! {
      val = some_operation() => {
        let _ = tx.send(val);
      }
      _ = tx.closed() => {
        // `some_operation()` is canceled, the
        // task compeltes and `tx1` is dropped
      }
    }
  });
  
  tokio::spawn(async {
    let _ = tx2.send("two");
  });
  
  tokio::select! {
    val = rx1 => {
      println!("rx1 completed first with {:?}", val);
    }
    val = rx2 => {
      println!("rx2 completed first with {:?}", val);
    }
  }
}
```

## The Future Implemention

为了更好的理解 `select!` 是如何工作的，我们来假设看看其 `Future` 的实现会像什么样子，在这个简单的版本中，`select!` 包含了一些额外功能，如随机的选择第一个调用 `poll` 的分支。

```rust
use tokio::sync::oneshot;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct MySelect {
  rx1: oneshot::Receiver<&'static str>,
  rx2: oneshot::Receiver<&'static str>,
}

impl Future for MySelect {
  type Output = ();
  
  fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
    if let Poll::ready(val) = Pin::new(&mut self.rx1).poll(cx) {
      println!("rx1 compelted first with {:?}", val);
      return Poll::Ready(());
    }
    
    if let Poll::ready(val) = Pin::new(&mut self.rx2).poll(cx) {
      println!("rx2 compelted first with {:?}", val);
      return Poll::Ready(());
    }
    
    Poll::Pending
  }
}

#[tokio::main]
async fn main() {
  let (tx1, rx1) = oneshot::channel();
  let (tx2, rx2) = oneshot::channel();
  
  // use tx1 and tx2
  
  MySelect {
    rx1,
    rx2
  }.await;
}
```

`MySelect` 包含了每个分支使用的 `Future`，当他被 `poll` 时，首先会尝试 `poll` 第一个分支，如果该分支的 `Future` 已完成，则处理该分支的代码，并且 `MySelect` 也进入完成状态，当 `.await` 接收到已完成 `Future` 的通知后，该 `Future` 就会被丢弃，这就会导致 `MySelect` 中的各个 `Future` 都被丢弃，因此另一个未完成的分支的 `Future` 在这时会被取消执行。

记住前一节我们提到的:

> 当一个 `Future` 返回 `Poll::Pending` 时，他要 确保 执行器能够在稍后的某个时间点收到通知，如果无法保证这点，该任务会无限的挂起导致无法运行。

在 `MySelect` 的实现中我们没有使用到 `Context` 参数，因为所需的 Waker 已经通过 `cx` 传递给了更内层的 `Future`，内层的 `Future` 在需要的时候会使用我们传递的 `Waker`，当他们返回 `Poll::Pending` 时，MySelect 也同样返回 `Poll::Pending`，因此 `MySelect` 满足 `Waker` 的要求。

## Syntax

`select!` 宏能够处理两个以上的分支，事实上当前的分支限制数是 64 个，每个分支的结构如下

```console
<pattern> = <async experssion> => <handler>,
```

当 `select!` 宏被执行时，所有的 `<async expression>` 被收集起来然后并行的执行，当其中一个表达式完成时，他的返回结果会检查是否满足 `<pattern>` 条件，如果满足，则其他分支的所有表达式都将被丢弃，然后当前这个分支的 `<handler>` 将会被执行，`<handler>` 表达式在执行时还能够访问到 `<pattern>` 得到的绑定数据。

使用 `<pattern>` 最简单的方式是提供一个变量名，这样在 `<async expression>` 满足时，对应的 `<handler>` 可以在其代码中通过该变量名访问到 `<async expression>` 执行的结果。这也是为什么我们在最初的例子中可以在 `<handler>` 中使用 `<pettern>` 定义的 val 变量。

如果 `<pattern>` 的定义跟表达式执行的结果类型不匹配，则其他的异步表达式会会继续执行知道下一个完成，然后下一个完成的逻辑跟当前这样一样，继续尝试匹配对应的 `<pattern>`。

因为 `select!` 能够接受任意的异步表达式，所以也可以定义更为复杂的表达式来作为 `<async expression>`。

下面我们使用 `oneshot::Channel` 的输出结果及 `TCP` 连接的创建作为 `select!` 的表达式

```rust
use tokio::net::TcpStream;
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
  let (tx, rx) = oneshot::channel();
  
  // Spawn a task that sends a message over the channel
  tokio::spawn(async move {
    tx.send("done").unwrap();
  });
  
  tokio::select! {
    socket = TcpStream::connect("localhost:3465") => {
      println!("Socket connected {:?}", socket);
    }
    msg = rx => {
      println!("received message first {:?}", msg);
    }
  }
}
```

下面的例子则是从 `oneshot` 接受信息，或是启动 `TcpListener` 接收连接的循环

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() io::Result<()> {
  let (tx, rx) = oneshot::channel();
  
  tokio::spawn(async move {
    tx.send(()).unwrap();
  });
  
  let mut listener = TcpListener::bind("localhost:3465").await?;
  
  tokio::select! {
    _ = async {
      loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move { process(socket) });
      }
      // Help the rust type interface out
      Ok::<_, io::Error>(())
    } => {}
    _ = rx => {
      println!("terminating accept loop");
    }
  }
  
  Ok(())
}
```

上面代码中接收 `socket` 的循环会一直运行到出错，又或者 `rx` 接收到了信息，`_` 符号用来表示我们并不在意异步表达式的返回值。

## Return Value

`tokio::select!` 宏会将以执行的 `<handler>` 的返回值作为其返回值，如

```rust
async fn computation1() -> String {
  // ...
}

async fn computation2() -> String {
  // ...
}

#[tokio::main]
async fn main() {
  let out = tokio::select! {
    res1 = computation1() => res1,
    res2 = computation() => res2,
  };
  
  println!("Got = {}", out);
}
```

因此，该宏会要求每一个分支的 `<handler>` 都返回相同的类型，如果不需要最终的执行结果，那明确的返回 `()` 是一个比较好的做法。

## Errors

通常情况下 `?` 操作符可以用来向上传播表达式的错误信息，接着我们看看 `?` 操作符在异步表达式中是如何工作的。在异步的表达式中我们同样可以使用 `?` 操作符来向上传播错误，这意味着异步表达式的返回结果需要是 `Result` 类型。在 `<handler>` 块中使用 `?` 语句能够让错误传播出 `select!` 表达式。让我们来看个例子

```rust
use tokio::net::TcpListener;
use tokio::sync::oneshot;
use std::io;

#[tokio::main]
async fn main() -> io::Result<()> {
  // setup rx oneshot channel
  
  let listener = TcpListener::bind("localhost:3465").await?;
  
  tokio::select! {
    res = async {
      loop {
        let (socket, _) = listener.accept().await?;
        tokio::spawn(async move { process(sokcet) });
      }
      
      Ok::<_, io::Error>(())
    } => {
      res?;
    }
    _ = rx => {
      println!("terminating accept loop");
    }
  }
  
  Ok(())
}
```

我们看到示例中使用了 `listener.accept().await?`，这一句的 `?` 操作符能够将错误信息从表达式中传播给 `<pattern>` ，也就是 res，在发生错误的情况下 `res` 将被设置为 `Err(_)`，紧接着在 `<handler>` 中再次对 `res` 使用了 `?` 操作符，因此对应的错误信息将被再次传播到 `main` 函数中。

## Pattern Maching

让我们回顾一下 `select!` 宏分支匹配的定义

```console
<pattern> = <async expression> => <handler>,
```

到目前为止，我们只在 `<pattern>` 中使用了变量绑定，实际上任意的 Rust 模式匹配都可以在 `<pattern>` 中使用，比如当我们在从 多个 MPSC Channel 中接收数据时，可能会这样写

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
  let (mut tx1, mut rx1) = mpsc::channel(128);
  let (mut tx2, mut rx2) = mpsc::channel(128);
  
  tokio::spawn(async move {
    // Do something w/ `tx1` and `tx2`
  });
  
  tokio::select! {
    Some(v) = rx1.recv() => {
      println!("Got {:?} from rx1", v);
    }
    Some(v) = rx2.recv() => {
      println!("Got {:?} from rx2", v);
    }
    else => {
      println!("Both channels closed");
    }
  }
}
```

在这个示例中，`select!` 表达式同时在等待 `rx1` 及 `rx2` 的消息，在 Channel 关闭时对 `rect()` 的调用会返回 `None`，该返回值与 `<pattern>` 的定义不匹配，因此该对应的分支将被关闭，然后 `select!` 表达式会继续等待剩余的其他分支。

我们还发现 `select!` 表达式包含了一个 `else` 分支，因为需要对 `select!` 表达式进行求值，但是在使用模式匹配时可能所有的模式都匹配不上，在这种情况下我们就需要使用 `else` 分支来帮助 `select!` 求值。

## Borrowing

在我们创建任务时，所创建任务的异步代码块必须持有他使用的数据，但 `select!` 并没有这个限制，每个分支的表达式可以对数据进行借用以及并发的进行操作，在 Rust 的借用规则中，多个异步表达式能够借用同一个不可变引用，或者一个一步表达式能够借用一个可变引用。

下面来看一些例子，我们同时的发送同一份数据到两个不同的 TCP 终端。


```rust
use tokio::io::AsyncWriteExt;
use tokio::net::TcpStream;
use std::io;
use std::net::SocketAddr;

async fn race(
    data: &[u8],
    addr1: SocketAddr,
    addr2: SocketAddr
) -> io::Result<()> {
    tokio::select! {
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr1).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        Ok(_) = async {
            let mut socket = TcpStream::connect(addr2).await?;
            socket.write_all(data).await?;
            Ok::<_, io::Error>(())
        } => {}
        else => {}
    };

    Ok(())
}
```

示例中 `data` 这个变量以不可变的形式被不同的异步分支借用，当其中一个分支的操作成功执行后，另一个分支就会被丢弃。因为匹配的模式需要是 `Ok(_)`，所以当其中一个分支匹配失败，另一个会继续执行下去。

当引用出现在了每个分支的 `<handler>` 中，`select!` 保证只会有一个 `handler` 会执行，因此各个不同的 `<handler>` 能够对同一份数据都持有可变的借用。

比如下面在多个分支中对 out 变量进行了修改

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx1, rx1) = oneshot::channel();
    let (tx2, rx2) = oneshot::channel();

    let mut out = String::new();

    tokio::spawn(async move {
        // Send values on `tx1` and `tx2`.
    });

    tokio::select! {
        _ = rx1 => {
            out.push_str("rx1 completed");
        }
        _ = rx2 => {
            out.push_str("rx2 completed");
        }
    }

    println!("{}", out);
}
```

## Loops

`select!` 宏常常会被在循环中使用，在这一小节我们来看一些将 `select!` 用在循环中的常见例子。首先从从多个不同的 Channel 读取数据开始

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx1, mut rx1) = mpsc::channel(128);
    let (tx2, mut rx2) = mpsc::channel(128);
    let (tx3, mut rx3) = mpsc::channel(128);

    loop {
        let msg = tokio::select! {
            Some(msg) = rx1.recv() => msg,
            Some(msg) = rx2.recv() => msg,
            Some(msg) = rx3.recv() => msg,
            else => { break }
        };

        println!("Got {}", msg);
    }

    println!("All channels have been closed.");
}
```

这个示例尝试从三个 Channel 中读取数据，只要任意一个 Channel 接收到了数据，就将他输出到 STDOUT 中，当 `rect()` 函数返回 `None` 时，因为有模式匹配的保护，这个循环会继续执行， `select!` 宏会继续等待剩余的 Channel。当所有的 Channel 都关闭后，`else` 分支会被执行然后循环被中止。

`select!` 宏会随机的选择可读的分支，在上例中当多个 Channel 中都有可读的数据时，将随机选择一个 Channel 来读取。这个实现是为了处理循环中消费消息的能力落后于生产消息这个场景所带来的问题，这个场景意味着 Channel 总会被填满，如果 `select!` 没有随机的选取分支，将导致循环中的 `rx1` 永远是第一个检查是否有数据可读的分支，如果 `rx1` 一直都有新的消息要处理，那其他分支中的 Channel 将永远不会被消费。

> 如果 `select!` 被求值时，其中的多个 Channel 都存在排队中的消息，只有一个 Channel 的消息会被消费，其他所有的 Channel 都不会进行任何检查，他们的消息会被一直存在 Channel 中，直到循环的下一轮迭代，这些消息并不会丢失。

## Resuming an async operation

现在我们来展示如果在多次 `select!` 调用中执行同一个异步操作，在这个例子中，我们有一个类型是 `i32` 的 Channel，还有一个异步的函数，我们希望在这个异步函数完成或者是我们的 Channel 接收到偶数时推出循环。

```rust
async fn action() {
    // Some asynchronous logic
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);    
    
    let operation = action();
    tokio::pin!(operation);
    
    loop {
        tokio::select! {
            _ = &mut operation => break,
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    break;
                }
            }
        }
    }
}
```

在示例中看到，我们并没有把对 `action()` 的调用放到 `select!` 的某个分支，相反，我们在循环前先调用了 `action()` 并将其返回值保存到了一个名为 `operation` 的变量中，并且 没有在他上面调用 `.await`，然后以 `operation` 作为参数调用了 `tokio::pin!` 宏。

在 `select!` 循环中，我们使用了 `&mut operation` 而不是直接使用 `operation`。这个 `operation` 变量在整个异步的操作中都存在，每次循环的迭代都会使用同一个 `operation` 而不是每次都调用一次 `action()`。

另外一个 `select!` 的分支则会从 Channel 中接收消息，如果接收到的消息是偶数，则完成这个循环，否则进入下一次循环继续执行 `select!`。

这是我们第一次使用 `tokio::pin!`，我们现在先不深入的去纠结 Pinning 的细节，唯一要了解的就是，想要在一个变量的引用上调用 `.await`，这个引用的变量必须是 `Pinned` 或者需要实现 `Unpin`。

如果我们移除 `tokio::pin!` 这一行代码，将会得到下面的错误

```console
error[E0599]: no method named `poll` found for struct
     `std::pin::Pin<&mut &mut impl std::future::Future>`
     in the current scope
  --> src/main.rs:16:9
   |
16 | /         tokio::select! {
17 | |             _ = &mut operation => break,
18 | |             Some(v) = rx.recv() => {
19 | |                 if v % 2 == 0 {
...  |
22 | |             }
23 | |         }
   | |_________^ method not found in
   |             `std::pin::Pin<&mut &mut impl std::future::Future>`
   |
   = note: the method `poll` exists but the following trait bounds
            were not satisfied:
           `impl std::future::Future: std::marker::Unpin`
           which is required by
           `&mut impl std::future::Future: std::future::Future`
```

这个错误信息并不是很清晰，当然也有我们还没深入了解 `Future` 的原因。当前的话只需要了解到，每个想要在其上调用 `.await` 的值都需要实现 `Future` ，如果你遇到提示说尝试在一个没有实现 Future 的引用中调用 `.await` 的错误的话，那大概率就是需要去 `Pinned` (钉住) 这个 `Future` 了。

更多关于 Pin 的信息可以从 [标准库](https://doc.rust-lang.org/std/pin/index.html) 文档中了解。

## Monifiying a branch

接下来看一个略微复杂一点的循环，我们有

1. 一个 i32 类型的 Channel
2. 一个用来使用这个 i32 值的异步函数

我们想实现的逻辑是

1. 等待 Channel 中收到一个偶数
2. 使用这个偶数来调用这个异步函数
3. 等待这个异步函数，与此同时处理更多的来自于 Channel 的数字
4. 如果新的偶数在这个异步函数完成前收到，停止这个异步的函数，然后用新的偶数来启动这个异步函数

```rust
async fn action(input: Option<i32>) -> Option<String> {
    // If the input is `None`, return `None`.
    // This could also be written as `let i = input?;`
    let i = match input {
        Some(input) => input,
        None => return None,
    };
    // async logic here
}

#[tokio::main]
async fn main() {
    let (mut tx, mut rx) = tokio::sync::mpsc::channel(128);
    
    let mut done = false;
    let operation = action(None);
    tokio::pin!(operation);
    
    tokio::spawn(async move {
        let _ = tx.send(1).await;
        let _ = tx.send(3).await;
        let _ = tx.send(2).await;
    });
    
    loop {
        tokio::select! {
            res = &mut operation, if !done => {
                done = true;

                if let Some(v) = res {
                    println!("GOT = {}", v);
                    return;
                }
            }
            Some(v) = rx.recv() => {
                if v % 2 == 0 {
                    // `.set` is a method on `Pin`.
                    operation.set(action(Some(v)));
                    done = false;
                }
            }
        }
    }
}
```

我们使用了跟上一个示例类似的策略，异步的函数在循环外调用并将返回值赋给 `operation` `变量，Pinned` 这个变量。循环会从 `operation` 跟 Channel 中做 `select!`。

我们看到 `action` 函数接收的是 `Option<i32>` 类型的参数，在从 Channel 收到偶数之前，我们需要初始化 `operation`，因此我们让 `action` 接收 `Option` 参数并以 `Option` 作为返回类型，如果传递的是 None 参数，他会直接返回 `None`，在循环的一开始 `operation` 会马上完成并返回 `None`。

这个示例还使用了一些新的语法，在第一个分支中包含了 , `if !done` 语句，这称为分支的先决条件，在解释他用来干什么之前，我们看看移除这个先决条件会发生什么

```console
thread 'main' panicked at '`async fn` resumed after completion', src/main.rs:1:55
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

这个错误发生在我们尝试对已完成的 `operation` 重复调用 `.await` 时，通常来说，在调用 `.await` 之后对应的值是已经被消费掉的。在我们的例子中，我们是对引用调用了 `.await` ，这意味着 `operation` 在完成后仍在循环中被调用。

为了避免这个问题导致的崩溃，我们需要小心的避免当 `operation` 完成时再对其进行调用，因此使用了 `done` 变量来记录 `operation` 的完成状态。`select!` 分支提供了先决条件这个机制，先决条件在 `select!` 调用对应分支 `.await` 之前进行检查，如果先决条件的返回值是 `false` 则该分支在当前 `select!` 中被禁用。我们把 `done` 变量初始化为 `false`，只有当 `operation` 完成时他会被设置为 `true`。因此在接下来的循环中对 `operation` 进行检查的分支会被禁用，然后当从 Channel 中接收到新的偶数时，`operation` 会被重置，然后 `done` 会被设置为 `false`。

## Per-Task Concurrency

`tokio::spawn` 跟 `select!` 都同样会并发的执行其中的异步操作，但是他们使用的策略是不同的。`tokio::spawn` 是接收异步的操作然后创建一个新的任务去运行他，任务是 Tokio 运行时调度的基本单位，两个不同的任务会被 Tokio 独立的进行调度，他们可能会同时被调度到不同的操作系统线程上执行，因此创建一个新任务跟创建新的线程有着相同的限制: 没有借用。

`select!` 使用同一个任务来运行所有的分支，因为所有在 `select!` 中的分支都在同一个任务中执行，所以他们永远不会同时的触发，`select!` 宏让单个任务能够多路复用多个异步的操作。