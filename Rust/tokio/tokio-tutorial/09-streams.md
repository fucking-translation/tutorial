# Streams

[原文](https://tokio.rs/)

</br>

Stream 表示一个异步的数据序列，我们用 `Stream Trait` 来表示跟标准库的 `std::iter::Iterator` 类似的概念，只是他是异步的。Stream 可以使用异步的函数来迭代，他们也可以使用其他的适配器来进行一些转换，Tokio 提供了一些常用的适配器，他们都实现在了 `StreamExt Trait` 上。

Tokio 提供的 Stream 支持是通过一个独立的包来实现的，他就是 `tokio-stream`

```toml
tokio-stream = "0.1"
```

> 当前 Tokio 的 Stream 工具库都在 `tokio-stream` 包中，只要 Stream Trait 在 Rust 的标准库中稳定发布了，Tokio 的 Stream 工具库就会移进 `tokio` 包中。

## Iteration

到目前为止，Rust 这门编程语言尚未支持异步的循环，因此要对 Stream 进行迭代我们需要用到 `while let` 循环及 `StreamExt::next()`.

```rust
use tokio_stream::StreamExt;

#[tokio::main]
async fn main() {
  let mut stream = tokio_stream::iter(&[1, 2, 3]);
  
  while let Some(v) = stream.next().await {
    println!("GOT = {:?}", v);
  }
}
```

跟迭代器一样，`next()` 函数返回 `Option<T>` 类型，其中 `T` 是 Stream 类型迭代时返回值的类型，`None` 则表示该 Stream 的迭代已经终止。

### Mini-Redis Broadcast

现在来看一个略微复杂的 Mini-Redis 客户端的例子。

完整的代码可以看 [这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/streams/src/main.rs)。

```rust
use tokio_stream::StreamExt;
use mini_redis::client;

async fn publish() -> mini_redis::Result<()> {
  let mut client = client::connect("127.0.0.1:6379").await?;
  
  // Publish some data
  client.publish("numbers", "1".into()).await?l
  client.publish("numbers", "two".into()).await?l
  client.publish("numbers", "3".into()).await?l
  client.publish("numbers", "four".into()).await?l
  client.publish("numbers", "five".into()).await?l
  client.publish("numbers", "6".into()).await?l
}

async fn subscribe() -> mini_redis::Result<()> {
  let client = client::connect("127.0.0.1:6379").await?;
  let subscriber = client::subscribe(vec!["numbers".to_string()]).await?;
  let messages = subscriber.into_stream();
  
  tokio::pin!(messages);
  
  while let Some(msg) = messages.next().await {
    println!("Got = {:?}", msg);
  }
  
  Ok(())
}

#[tokio::main]
async fn main() -> mini_redis::Result<()> {
  tokio::spawn(async {
    publish().await
  });
  
  subscribe().await?;
  println!("DONE");
  
  Ok(())
}
```

在上面的代码中我们创建了一个用 Mini-Redis 在 `numbers` 频道中发布消息的任务，而在主任务中，我们订阅了 `number` 频道，并且每次在收到该频道的消息时将它打印了出来。

在订阅之后，我们在订阅者上面调用了 `into_stream()` 函数，这个函数消费了 `Subscriber` 然后返回了一个 在接收到消息时迭代数据的 Stream，我们还看到代码中使用了 `tokio::pin!` 宏来将 Stream 钉到了栈中。调用 `next()` 函数需要这个 Stream 是被 [Pinned 钉住](https://doc.rust-lang.org/std/pin/index.html) 的，而 `into_stream()` 函数所返回的 Stream 是未 `Pin` 的，因此我们必须将其 `Pin` 住才能进行迭代。

> 一个 Rust 的值被 `Pin` 之后意味着他将不会在内存中被移动，一个被 `Pin` 的值所具有的一个核心属性是：调用者能够安全的获取其中的指针信息，并且其中的指针信息必定是有效的。这个特性是用来为 `async/await` 中跨多个 `.await` 调用的实现提供支持的。

如果我们忘了将 Stream Pin 住，将得到类似下面的错误信息

```console
error[E0277]: `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>` cannot be unpinned
  --> streams/src/main.rs:29:36
   |
29 |     while let Some(msg) = messages.next().await {
   |                                    ^^^^ within `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`, the trait `Unpin` is not implemented for `from_generator::GenFuture<[static generator@Subscriber::into_stream::{closure#0} for<'r, 's, 't0, 't1, 't2, 't3, 't4, 't5, 't6> {ResumeTy, &'r mut Subscriber, Subscriber, impl Future, (), std::result::Result<Option<Message>, Box<(dyn std::error::Error + Send + Sync + 't0)>>, Box<(dyn std::error::Error + Send + Sync + 't1)>, &'t2 mut async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't3)>>>, async_stream::yielder::Sender<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't4)>>>, std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 't5)>>, impl Future, Option<Message>, Message}]>`
   |
   = note: required because it appears within the type `impl Future`
   = note: required because it appears within the type `async_stream::async_stream::AsyncStream<std::result::Result<Message, Box<(dyn std::error::Error + Send + Sync + 'static)>>, impl Future>`
   = note: required because it appears within the type `impl Stream`
   = note: required because it appears within the type `tokio_stream::filter::_::__Origin<'_, impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>`
   = note: required because it appears within the type `tokio_stream::map::_::__Origin<'_, tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>`
   = note: required because it appears within the type `tokio_stream::take::_::__Origin<'_, tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
   = note: required because of the requirements on the impl of `Unpin` for `tokio_stream::take::Take<tokio_stream::map::Map<tokio_stream::filter::Filter<impl Stream, [closure@streams/src/main.rs:22:17: 25:10]>, [closure@streams/src/main.rs:26:14: 26:40]>>`
```

当你遇到这样的错误信息时，记得将所需的值 Pin 住！

在我们尝试来运行这段代码前，先把 Mini-Redis 服务启动起来

```console
$ mini-redis-server
```

然后运行我们的代码，将会得到如下的输出

```console
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"four" })
got = Ok(Message { channel: "numbers", content: b"five" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

在订阅跟发布之间产生竟态条件时，最前面的部分的数据可能会丢失，这是有 `Pub/Sub` 机制决定的。这个程序是不会退出的，因为在 Mini-Redis 的订阅中只要服务端没有退出，订阅的程序就会一直运行下去。

下面我们来看看如何扩展我们的这个程序。

## Adapters

一些接收 `Stream` 后返回另外一种 `Stream` 的函数我们将它成为 `Stream` 适配器，他们来自于称为 **适配器模式** 的设计模式，常见的适配器包括如 `map`、`take` 及 `filter` 等。

接下来我们来更新我们的 Mini-Redis 客户端让他在收到三个消息后能够终止迭代，退出程序。我们使用 `take` 来完成这个功能，这个适配器限制了 Stream 能够返回最多 `n` 个消息


```rust
let message = subscriber
    .into_stream()
    .take(3);
```

然后再次执行程序，我们会得到下面的结果

```console
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"two" })
got = Ok(Message { channel: "numbers", content: b"3" })
```

这次程序能够自动退出了。

接下来，我们来限制 Stream 只接收单个字符的数字，我们将通过检查消息的长度来实现这个目的，因此我们将使用 `filter` 适配器来过滤掉那些不满足条件的消息

```rust
let messages = subscriber
    .into_stream()
    .filter(|msg| match msg {
    Ok(msg) if msg.content.len() == 1 => true,
    _ => false,
    })
    .take(3);
```

再次运行程序我们将得到下面的输出

```console
got = Ok(Message { channel: "numbers", content: b"1" })
got = Ok(Message { channel: "numbers", content: b"3" })
got = Ok(Message { channel: "numbers", content: b"6" })
```

在上面的代码还需要注意的是，适配器间不同的使用顺序是表示了不同意义的，先调用 `filter` 在调用 `take` 跟先调用 `take` 后调用 `filter` 是不同的。

最后，我们来去掉最后输出信息的 `Ok(Message { ... })` 部分，只保留具体的值，这次我们会使用 `map`，因为他是应用在 `filter` 之后的，因此传递给 `map` 的数据都是 `Ok`，我们可以放心的调用 `unwrap()`。

```rust
let messager = subscriber
    .into_stream()
    .filter(|msg| match msg {
  	Ok(msg) if msg.content.len() == 1 => true,
    _ => false
    })
    .map(|msg| msg.unwrap().content)
    .take(3);
```

然后得到下面的输出

```console
got = b"1"
got = b"3"
got = b"6"
```

另外一个可供选择的方式是使用另一个组合了 `filter` 跟 `map` 的，一个叫 `filter_map` 的适配器。

更多可用的适配器，可以在 [这里](https://docs.rs/tokio-stream/0.1/tokio_stream/trait.StreamExt.html) 找到。

## Implementing Stream

`Stream Trait` 跟 `Future Trait` 非常相似

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Stream {
  type Item;
  
  fn poll_next(
    self: Pin<&mut self>,
    cx: &mut Context<'_>
  ) -> Poll<Option<Self::Item>>;
  
  fn size_hint(&self) -> (usize, Option<usize>) {
    (0, None)
  }
}
```

`Stream::poll_next()` 函数跟 `Future::poll` 很类似，区别在于 Stream 能够通过重复的调用来获取多个返回结果，正如我们在 `Async in depth` 中提到的，当 Stream 还未就绪时，他会返回 `Poll::Pending`，这时 任务的 `Waker` 会被注册起来，当 Stream 可以被再次 `Poll` 时，该 `Wake` 将会受到通知。

`size_hint()` 函数则是实现了跟迭代器的对应函数一样的功能。

通常来说，手动的来实现一个 Stream ，基本都是使用组合其他的 `Future` 跟 Stream 的方式，作为一个示例，我们使用在 `Async in depth` 中实现的 `Delay` ，将他转换为一个 Stream, 该 Stream 每个 10 毫秒的间隔会返回一次 `()`，一共会返回三次。

```rust
use tokio_stream::Steram;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::Duration;

struct Interval {
  rem: usize,
  delay: Delay,
}

impl Stream for Interval {
  type Item = ();
  
  fn poll_next(mut self: Pin<&mut Self>, cx: &mut Context<'_>)
  	-> Poll<Option<()>> {
      if self.rem == 0 {
        return Poll::Ready(None);
      }
      
      match Pin::new(&mut self.delay).poll(cx) {
        Poll::Ready(_) => {
          let when = self.delay.when + Duration::from_millis(10);
          self.delay = Delay { when };
          self.rem -= 1;
          Poll::Ready(Some(()))
        }
        Poll::Pending => Poll::Pending,
      }
  }
}
```

### async-stream

手动的来实现 `Stream` 是一件乏味的事，不幸的是 Rust 语言现在还不支持用 `async/await` 的语法来定义 Steram，这个特性正在实现中，但还没完成。

`async-stream` 包提供了一个临时的解决方案，这个包提供了一个 `async_stream!` 宏来将输入转换为 Stream, 使用这个包上面的 `Interval` 例子能够写成

```rust
use async_stream::stream;
use std::time::{Duration, Instant};

stream! {
  let mut when = Instant::new();
  for _ in 0..3 {
    let delay = Delay { when };
    delay.await;
    yield ();
    when += Duration::from_millis(10);
  }
}
```