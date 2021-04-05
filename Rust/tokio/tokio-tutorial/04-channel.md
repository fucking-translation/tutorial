# Channel

[原文](https://tokio.rs/)

</br>

现在开始来学一些 Tokio 中的并发支持。开始把这些并发的东西应用到我们的客户端中，比如我们想要同时运行两个 Redis 的命令时，可以为每个命令创建一个任务，这样两个命令就能够并行的执行了。

首先我们来简单的尝试一下

```rust
use mini_redis::{client, Result};

#[tokio::main]
async fn main() {
    let mut client = client::connect("127.0.0.1:6379").await?;
    
    let t1 = tokio::spawn(async {
        let res = client.get("hello").await?;
    });
    
    let t2 = tokio::spawn(async {
        client.set("hello", "world".into()).await?;
    }):

    t1.await.unwrap();
    t2.await.unwrap();   
}
```

因为`Client`没有实现`Copy`，并且两个任务都同时需要在其中使用到`client`变量，所以是编译不过的。并且，因为 `Client::set`需要使用`&mut self`也就是可变引用作为参数，因此该对象的使用实际上是排他的。我们可以为每个连接创建一个任务，但那并不是个好主意，我们不能够使用`std::sync::Mutex`因为会有持有锁跨越`.await`的情形；我们不能使用`tokio::sync::Mutex`，那会导致在同一时刻只有一个请求在处理。如果`client`实现了 [pipelining](https://redis.io/topics/pipelining)，那异步的`Mutex`就会无法充分的利用当前连接了。

## Message Passing

最好的方式是使用消息传递，这种方式需要创建一个单独的任务来管理`client`资源，任何一个想要发送命令的任务都需要发送消息给管理`client`的任务，该任务会处理收到的命令然后将处理结果回复给请求的任务。

使用这个策略，可以只创建一个连接，管理`client`的任务就可以有序的处理`get`跟`set`请求了，而`Channel`则相当于一个缓冲，就算`client`处于繁忙状态其他的任务页可以发送命令到 `Channel`，当他能够处理新请求的时候，他会从`Channel`中获取下一个请求进行处理，这样的方式能够带来很好的吞吐量，还可以再将其扩展为使用连接池的方式。

## Tokio's Channel Primitives

Tokio 提供了数种用于处理不同场景的`Channel`

- `mpsc`: 多生产者、单消费者的`Channel`，能够发送多个信息
- `oneshot`: 单生产者、单消费者的`Channel`，只能发送一个信息
- `broadcast`: 多生产者、多消费者，能够发送多个信息，每个消费者都能收到所有信息
- `watch`: 单生产者、多消费者，能够发送多个信息，但不会保存历史信息，消费者只能收到最新的信息

如果需要多生产者、多消费者的`Channel`但希望每个信息只被一个消费者收到，可以使用[async-channel](https://docs.rs/async-channel/) 包。还有其他的一些不能用在 Rust 的异步编程中的`Channel`实现，比如`std::sync::mpsc`跟 `crossbeam::channel`。这些`Channel`以堵塞线程的方式等待信息的到来，所以不能够在异步的代码中使用。

在这一节中，我们会用到`mpsc`跟`oneshot`，其他的`Channel`类型会在后续的章节中用到，然后，本章完整的代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs)找到。

## Define The Message Type

在大部分使用消息传递的场景中，负责处理消息的任务都需要响应不止一种命令。在我们的案例中，该任务需要响应`GET`跟`SET`两种命令，因此我们首先会定义一个包含所有命令类型的`Command`枚举。

```rust
use bytes::Bytes;

#[derive[Debug]]
enum Command {
  Get {
    key: String
  },
  Set {
    key: String,
    value: Bytes,
  }
}
```

## Create The Channel

然后在`main`函数中创建一个`mpsc`类型的`Channel`:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
  // Create a new channel with a capacity of at most 32
  let (tx, mut rx) = mpsc::channel(32);
}
```

`mpsc`的`Channel`将用来发送命令给管理`Redis`连接的任务，其多生产者的模式允许多个任务通过他来发送消息。创建`Channel`的函数返回了两个值，一个发送者跟一个接收者，这两个句柄通常是分开使用的，他们会被移到到不同的任务中。

创建`Channel`时设置了容量为 32，如果消息发送的速度超过了接收的速度，这个`Channel`只会最多保存 32 个消息，当其中保存的消息超过了 32 时，继续调用`send(...).await`会让发送的任务进入睡眠，直到接收者又从`Channel`中消费了消息。

在使用中会通过`clone`发送者的方式，来让多个任务同时发送消息，如下例:

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() -> Result<()> {
    let (tx, mut rx) = mpsc::channel(32);
    let tx2 = tx.clone();

    tokio::spawn(async move {
        tx.send("sending from first handle").await;
    });

    tokio::spawn(async move {
        tx2.send("sending from second handle").await;
    });

    while let Some(message) = rx.recv().await {
        println!("GOT = {}", message);
    }

    Ok(())
}
```

每个消息最后都会发送给唯一的接收者，因为通过`mpsc`创建的接收者是不能`clone`的。

当所有发送者出了自身的作用域或被`drop`后就不再允许发送消息了，在这个时候接收者会返回`None`，意味着所有的发送者已经被销毁，所以`Channel`也已经被关闭了。

在我们的示例中，`Redis`的连接是管理任务所负责的，他知道可以在管理的`Channel`都关闭后，`Redis`的连接就不会再有人使用了，因此可以关闭`Redis`的连接了。

## Spawn Manager Task

接下来创建负责处理来自`Channel`的任务，首先创建连接到 `Redis`的客户端对象，然后依次接收信息并调用`Redis`去处理。

```rust
use mini_redis::client;

let manager = tokio::spawn(async move {
  let mut client = client::connect("127.0.0.1:6379").await.unwrap();

  while let Some(cmd) = rx.recv().await {
    use Command::*;
    match cmd {
      Get { key } => {
        client.get(&key).await;
      }
      Set { key, value } => {
        client.set(&key, value).await;
      }
    }
  }
});
```

然后更新之前的两个任务，将直接使用`Redis`连接的方式改为通过 `Channel`发送命令。

```rust
let tx2 = tx.clone();
let t1 = tokio::spawn(async move {
  let cmd = Command::Get {
    key: "hello".to_string()
  };
  tx.send(cmd).await.unwrap();
});

let t2 = tokio::spawn(async move {
  let cmd = Command::Set {
    key: "foo".to_string(),
    value: "bar".into()
  };
  tx2.send(cmd).await.unwrap();
});
```

然后在`main`函数的最下面，在程序退出前我们调用前面定义的几个 JoinHandle (t1、t2、manager) 的`.await`。

## Receive Responses

最后一步是需要接收`manager`任务对我们请求的响应。在操作成功的情形`GET`命令需要返回我们之前调用`SET`的结果。

我们将通过传递`oneshot`类型的`Channel`来获取响应，`oneshot`是单生产者、单消费者的`Channel`，他还为只传递单次消息做了优化，在我们的示例中，这个单次消息就是我们所需的响应。

跟`mpsc`类似，`oneshot::channel()`返回发送者跟接收者。

```rust
use tokio::sync::oneshot;
let (tx, rx) = oneshot::channel();
```

而跟`mpsc`不同的是他不需要定义容量，因为他的容量永远都为 1，还有返回的发送者及接收者都不能够进行`clone`。

为了接收来自`manager`任务的响应，在发送命令之前我们需要先创建好`oneshot`实例，发送者的部分会被包含到命令之中，以便`manager`用来发送响应，接收者则由任务自己用来接收响应。

首先我们先来更新`Commoand`的定义以让他包含发送者类型`Sender`。为了书写方便我们为`Sender`定义了别名。

```rust
use tokio::sync::oneshot;
use bytes::Bytes;

type Responder<T> = oneshot::Sender<mini_redis::Result<T>>;

#[derive(Debug)]
enum Command {
  Get {
    key: String,
    resp: Responder<Option<Bytes>>,
  },
  Set {
    key: String,
    value: Vec<u8>,
    resp: Responder<()>,
  }
}
```

接着，更新发送命令的部分，让他包含`oneshot::Sender`。

```rust
let tx2 = tx.clone();
let t1 = tokio::spawn(async move {
  let (resp_tx, resp_rx) = oneshot::channel();
  let cmd = Command::Get {
    key: "hello".to_string(),
    resp: resp_tx,
  };
  tx.send(cmd).await.unwrap();

  let res = resp_rx.await;
  println!("GOT = {:?}", res);
});

let t2 = tokio::spawn(async move {
  let (resp_tx, resp_rx) = oneshot::channel();
  let cmd = Command::Set {
    key: "foo".to_string(),
    value: "bar".into(),
    resp: resp_tx,
  };
  tx2.send(cmd).await.unwrap();

  let resp = resp_rx.await.unwrap();
  println!("GOT = {:?}", res);
});
```

最后，更新`manager`任务让他通过`oneshot`的`Channel`返回最终的响应。

```rust
let manager = tokio::spawn(async move {
  let mut client = client::connect("127.0.0.1:6379").await.unwrap();

  while let Some(cmd) = rx.recv().await {
    use Command::*;
    match cmd {
      Get { key, mut resp } => {
        let res = client.get(&key).await;
        let _ = resp.send(res);
      }
      Set { key, value, resp } => {
        let res = client.set(&key, value.into()).await;
        let _ = resp.send(res);
      }
    }
  }
})
```

调用`oneshot::Sender`的`send`会立即返回结果因此无需再调用`.await`，这是因为`send`函数会在调用的时候立即返回成功或失败的结果。

通过`oneshot`发送的消息只有在接收者已经被销毁时返回错误，这表示已经没有接受者期待我们的响应了，并且接收者不再等待响应是一种可以接受的结果。因此发送者返回的`Err`可以不进行处理。

完整的代码可以在[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/channels/src/main.rs)找到。

## Backpressure And Bounded Channels

无论何时介绍并发或者队列，对其容量的限制都是很重要的，因为他能在系统优雅的处理负载，无限制的队列最终会把所有的可用内存都耗尽导致系统以不可预测的方式失效。

Tokio 小心的避免绝对的队列，其中最重要的一部分就是所有的异步操作都是`lazy`，考虑下面的代码

> `lazy`表示操作不会马上执行，只有在有需要的时候才执行

```rust
loop {
  async_op();
}
```

如果异步操作马上就执行，这个循环会不断的将`async_op`放到任务队列中去执行，而不管其之前的操作是否已经完成，这就体现为无限的队列，以回调形式或立即执行的`Future`异步系统很容易就会受这些操作影响。

然而，以 Rust 的异步编程机制实现的 Tokio 并不会真正的去执行上面代码片段中的`async_op`。这是因为 没有在他上面调用`.await`，如果将上面的代码改为使用`.await`，这个循环每次都会等待之前的任务完成后才开始一个新的任务。

```rust
loop {
  // Will not repeat until `async_op` completes
  async_op().await;
}
```

并行跟队列已经郑重的介绍了，当真正想要那么做的时候，可能会用到下面这些

- `tokio::spawn`
- `select!`
- `join!`
- `mpsc::channel`

在这么做的时候，要小心的确保这些操作都是有限的，比如在等待接收 TCP 连接的循环中，要确保能打开的套接字的上限。在使用`mpsc::channel`时要选择一个合理的容量，具体的合理值根据程序不同而不同。

小心的选择这些限额将能够大幅度的提高 Tokio 程序的可靠性。