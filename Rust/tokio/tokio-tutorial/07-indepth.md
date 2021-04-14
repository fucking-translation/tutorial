# Async In Depth

[原文](https://tokio.rs/)

</br>

现在，我们已经对 Rust 异步方式跟 Tokio 有了比较全面的了解，接着我们将继续更加深入的来挖掘 Rust 的异步运行时模型。在我们前面的教程中，我们提到了 Rust 使用了一种特殊的方式来实现异步的目标，现在我们来说说他到底是什么方式。

## Futures

现在让我们来看一个非常简单的异步函数作为整体的概览，这个函数相对于我们已经完成的教程并没有任何新的东西

```rust
use tokio::net::TcpStream;

async fn my_async_fn() {
  println!("hello from async");
  let _socket = TcpStream::connect("127.0.0.1:3000").await.unwrap();
  println!("async TCP operation complete");
}
```

然后我们来调用这个函数得到他的返回值，然后在返回值上调用 `.await`

```rust
#[tokio::main]
async fn main() {
  let waht_is_this = my_async_fn();
  // Nothing has been printed yet.
  what_is_this.await;
  // Text has been printed and socket has been
  // established and closed.
}
```

`my_async_fn()` 的返回值是一个 `Future`, 每个 `Future` 都会实现标准库提供的 `std::future::Future Trait`，他们表示了一个值，该值包含了进程内的异步计算。

`std::future::Future Trait` 的定义如下：

```rust
use std::pin::Pin;
use std::task::{Context, Poll};

pub trait Future {
  type Output;
  
  fn poll(self: Pin<&mut Self>, cx: &mut Context )
      -> Poll<Self::Output>; 
}
```

在 `Trait` 中定义的关联类型 `Output` 表示该 Future 在完成时会产生的结果，而 `Pin` 类型则是 Rust 用来支持异步操作中借用机制的基础。具体的细节可以从标准库的[文档](https://doc.rust-lang.org/std/pin/index.html)了解。

跟其他编程语言实现 `Future` 的方式不同，Rust 的 `Future` 并不表示任何跟后台执行相关的含义，他只表示了该计算原本的过程。持有 `Future` 的人需要负责的是通过调用 `Future::poll` 函数，来让该计算能够继续推进，该推进的过程则称为 `Polling`。

## Implementing Future

现在让我们来实现一个包含下面功能的简单的 `Future`

1. 等待指定的时间
2. 打印文本到标准输出
3. 返回一个字符串

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

struct Delay {
  when: Instant,
}

impl Future for Delay {
  type Output = &'static str;
  
  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
      -> Pool<&'static str> {
        if Instant::now() > self.when {
          println!("Hello world");
          Poll::Ready("done")
        } else {
          // Ignore this line for now
          cx.waker().wake_by_ref();
          Poll::Pending
        }
  }
}

#[tokio::main]
async fn main() {
  let when = Instant::now() + Duration::from_millis(10);
  let future = Delay { when };
  
  let out = future.await;
  asser_eq!(out, "done");
}
```

## Async fn as a Future

在 `main` 函数中，我们初始化了一个 `Future` 并调用了其 `.await`，在异步的函数中，我们可以在任何实现了 `Future Trait` 的值上调用 `.await`。实际上，调用一个标记为 `async` 的函数会返回一个匿名的实现了 `Future` 的值，在 `async fn main()` 函数中，该调用生成的 `Future` 类似下面的代码段

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

enum MainFuture {
  // Initialized, never polled
  State0,
  // Waiting on `Delay`, i.e. the `future.await` line
  State1(Delay),
  // The future has completed.
  Terminater,
}

impl Future for MainFuture {
  type Output = ();
  
  fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>)-> Poll<()> {
    use MainFutures::*;

    loop {
      match *self {
        State0 => {
          let when = Instant::now() + Duration::from_millis(10);
          let future = Delay { when };
          *self = State1(future);
        }
        State1(ref mut my_future) => {
          match Pin::new(my_future).poll(cx) {
            Poll::Ready(out) => {
              assert_eq(out, "done");
              *self = Terminated;
              return Poll::Ready(());
            }
            Poll::Pending => {
              return Poll::Pending;
            }
          }
        }
        Terminated => {
          panic!("future polled after completion")
        }
      }
    }
  }
}
```

Rust 的 `Futures` 被实现成了状态机，在上面的代码中 `MainFuture` 用一个枚举来表示该 `Future` 所具有的所有状态。一开始该 `Future` 被置为了 `State0`，当 `poll` 被调用时，该 `Future` 尝试让自己的状态进入下一步，即 `State1`，在这一状态中会继续尝试让自己内部的状态 (Delay) 也进入下一步。当该 `Future` 的异步操作完成时，他会将最终的返回结果与 `Poll::Ready` 一起返回。

如果该 `Future` 现在无法完成，这一般取决于其内部所依赖的其他资源是否完成，他会返回 `Poll::Pending`，调用者收到 `Poll::Pending` 便能够确认该 `Future` 现在无法继续推进，因此调用者会在稍后继续尝试调用 `poll` 函数来推进其状态。

我们通常会看到 `Future` 由其他的一些 `Future` 组成，调用最外层的 `Future` 会导致内部的 `Future` 的 `poll` 函数也被调用。

## Executors

异步的 Rust 函数会返回 `Future`，而 `Future` 需要有人来调用他的 `poll` 函数以推进他的状态，并且 `Future` 经常会由许多其他的 `Future` 组成，现在的问题就在于：谁来调用最外层 `Future` 的 `poll` 函数？

回顾之前信息，为了运行一个异步的函数，我们要么把他传递给 `tokio::spawn` 要么就给 `main` 函数标注上 `#[tokio::main]` 宏。该宏会将最终生成的 `Future` 传递给 `Tokio` 的执行器 (Executor)。执行器会负责来调用最外层的 `Future::poll` 函数，驱动整个异步操作的进行。

### Mini Tokio

为了更好的理解这些知识，并将他们都整合起来，我们来实现一个迷你的 Tokio。完整的代码可以从[这里](https://github.com/tokio-rs/website/blob/master/tutorial-code/mini-tokio/src/main.rs)找到。

```rust
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use futures::task;

fn main() {
    let mut mini_tokio = MiniTokio::new();

    mini_tokio.spawn(async {
        let when = Instant::now() + Duration::from_millis(10);
        let future = Delay { when };

        let out = future.await;
        assert_eq!(out, "done");
    });

    mini_tokio.run();
}

struct MiniTokio {
    tasks: VecDeque<Task>,
}

type Task = Pin<Box<dyn Future<Output = ()> + Send>>;

impl MiniTokio {
    fn new() -> MiniTokio {
        MiniTokio {
            tasks: VecDeque::new(),
        }
    }
    
    /// Spawn a future onto the mini-tokio instance.
    fn spawn<F>(&mut self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        self.tasks.push_back(Box::pin(future));
    }
    
    fn run(&mut self) {
        let waker = task::noop_waker();
        let mut cx = Context::from_waker(&waker);
        
        while let Some(mut task) = self.tasks.pop_front() {
            if task.as_mut().poll(&mut cx).is_pending() {
                self.tasks.push_back(task);
            }
        }
    }
}
```

上面会运行整个异步的代码， `Delay` 的实例会被创建并且等待其执行完成，只是现在的实现有一个很大的问题。当前的执行器在遇到未能完成的任务时不会挂起或进入睡眠，而是会不断的循环调用所有已创建的任务的 `poll`，但是在大部分的情况下，这些 `Future` 在返回 `Poll::Pending` 后并不会马上完成，继续的调用只会不断的返回 `Poll::Pending`，当前的实现会极大的消耗 `CPU` 资源。

理想情况下，我们希望 `mini-tokio` 只在有 `Future` 能够继续推进其状态时进行 `Poll`，这种情况发生在任务内部的某些资源操作完成时，比如一个任务希望从 `TCP` 套接字中读取数据，那就应该只在 `TCP` 套接字接收到数据的时候才进行 `poll`，在我们的案例中任务会被 `Instant` 实例堵塞，理想情况下，我们希望 `mini-tokio` 只在 `Instant` 指定的时间过去后才对任务进行 `Poll`。

为了实现这个目标，当一个资源在还没就绪的状态下被 `Poll` 时，他应该在自身就绪时发出通知。

### Wakers

到现在我们还没讨论到 `Waker`，他是资源在可用时用来通知那些等待中的任务的系统，接收到通知的任务意味着他们能够继续进行后续的操作。

让我们再来看一次 `Future::poll` 的定义

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context) -> Poll<Self::Output>;
```

`poll` 函数的 `Context` 类型参数中有个 `waker()` 函数，该函数会返回一个跟当前任务绑定的 `Waker`，这个 `Waker` 有一个 `wake()` 函数，调用这个函数就能够通知执行器，执行器会调度与该 `Wakier` 关联的任务，让他继续执行。资源在他就绪的时候就会调用 `wake()` 通知执行器让他继续对这个任务调用 `poll`。

### Update Delay

现在将 `Delay` 更新为使用 `Waker` 的方式

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};
use std::thread;

struct Delay {
  when: Instant,
}

impl Future for Delay {
  type Output = &'static str;
  
  fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<&'static str> {
    if Instant::now() >= self.when {
      println!("Hello world");
      Poll::Ready("done")
    } else {
      // Get a handle to the waker for the current task
      let waker = cx.waker().clone();
      let when = self.when;
      
      // Spawn a timer thread
      thread::spawn(move || {
        let now = Instant::now();
        
        if now < when {
          thread::sleep(when - now);
        }
        
        waker.wake();
      });
      
      Poll::Pending
    }
  }
}
```

现在，当等待的时间到达后，调用的任务会收到通知，然后执行器就能够确认该任务能够继续调度执行了。下一步我们要来更新 `mini-tokio` 让他来监听通知信息。

这里的 `Delay` 实现还是有一些问题，我们稍后会一起修复。

当一个 `Future` 返回 `Poll::Pending` 时，他要 确保 执行器能够在稍后的某个时间点收到通知，如果无法保证这点，该任务会无限的挂起导致无法运行。

返回 `Poll::Pending` 之后没有调用 `wake` 发送通知是一个很常见的错误。

回顾一下最初的 `Delay` 实现，下面是 `Future` 在 `Delay` 上的实现

```rust
impl Future for Delay {
    type Output = &'static str;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>)
        -> Poll<&'static str>
    {
        if Instant::now() >= self.when {
            println!("Hello world");
            Poll::Ready("done")
        } else {
            // Ignore this line for now.
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

在返回 `Poll::Pending` 之前，我们调用了 `cx.waker().wake_by_ref()`，这是为了满足实现 `Future` 的约定，因为我们需要负责向 `Waker` 发送通知。又因为我们没有专门实现一个定时器的管理线程，所以才直接在 `Poll::Pending` 前直接发送通知。调用之后的结果就是，我们的 `Future` 能够马上被重新调度，重新执行，虽然不代表能够马上完成任务、结束等待。

从上面的例子能够发现，不是只有在资源就绪的时候才能发送通知给 `Waker`，我们在不能够继续执行下去的状态仍然发送了通知给 `Wakter`，除了消耗更多的 `CPU` 以外这并不会造成什么问题，因为他只是导致了更多的循环调用。

### Updating Mini Tokio

接下来我们要更新 Mini Tokio 的实现，让他能够接收到消息通知，我们希望执行器只有在任务就绪的情况才去执行他，因此 Mini Tokio 将会提供自己的 `Waker` 实现。当 `Waker` 被唤醒时，需要被执行的任务会被先排到队列中等待，然后 Mini Tokio 在 `Poll` 这个任务的时候会把对应的 `Waker` 传递给他。

更新后的 Mini Tokio 会使用 `Channel` 来存储调度的任务，`Channel` 可以让来自不同线程的任务在其中排队，因此 `Waker` 必须要实现 `Send` 跟 `Sync` 才能够在不同的线程中传递，才能够在我们即将使用的 `crossbeam` 包中的 `Channel` 中使用，不使用标准库的是因为标准库中的 `Channel` 是非 `Sync` 的。

> `Sync` 跟 `Send Trait` 是用在 Rust 并发编程中的标志性的 Trait。一个实现了 `Send` 的类型说明他能够在不同的线程中被传递，大部分的类型都实现了 `Send`，当然也有少部分如 `Rc` 这些是没实现的。`Sync` 则说明类型能够被并行的访问其不可变引用。一个类型可以是 `Send` 但不是 `Sync` 的，比如 `Cell`，他可以使用不可变引用来进行修改，但不能够并发的被访问。
> 想要了解更多的细节的话，可以看 [Rust Book](https://doc.rust-lang.org/book/ch16-04-extensible-concurrency-sync-and-send.html) 的这个章节。

接着添加新的依赖，让我们能够使用所需的 Channel。

```rust
crossbeam = "0.7"
```

然后更新 Mini Tokio 的结构定义

```rust
use crossbeam::channel;
use std::sync::Arc;

struct MiniTokio {
  scheduled: channel::Receiver<Arc<Task>>,
  sender: channel::Sender<Arc<Task>>,
}

struct Task {
  // This will be filled in soon.
}
```

`Waker` 是 `Sync` 并且能够被 `Clone` 的，当 `wake` 被调用时，当前的任务必须能够被重新调度去执行，为了实现这点，我们定义了一个 `Channel`，当 `wake()` 被调用时，这个任务被 `Channel` 的 `Sender` 发送出去。接着我们的 `Task` 任务类型会实行具体的唤醒逻辑，为了实现这点，`Task` 需要包含创建的 `Future` 以及 `Channel` 的 `Sender`。

```rust
use std::sync{Arc, Mutex};

struct Task {
  // The `Mutex` is to make `Task` implement `Sync`. Only
  // one thread accesses `future` at any given time. The
  // `Mutex` is not required for correctness. Real Tokio
  // does not use a mutex here, bu real Tokio has more
  // lines of code than can fit in a single tutorial page.
  future: Mutex<Pin<Box<dyn Future<Output=()> + Send>>>,
  executor: channel::Sender<Arc<Task>>,
}

impl Task {
  fn schedule(self: &Arc<Self>) {
    self.executor.send(self.clone());
  }
}
```

为了调度该任务， 被 `Arc` 包装着的 `Task` 被发送 `Clone`后发送到了 `Channel` 中，接着我们需要来实现对 `std::task::Waker` 的调度处理了。标准库为此提供了较为底层的 [RawWakerVTable](https://doc.rust-lang.org/std/task/struct.RawWakerVTable.html) 的接口，这个做法为实现者提供了最大的灵活度，但同时也要求我们去实现大量的非安全代码，在这里我们使用了由 `futures` 包提供的 `ArcWake` 类型来替代复杂的 `RawWakerVTable` ，他让我们只需要简单的实现该 `Trait` 的方法，就能让 `Task` 成为一个 `Waker`。

首先到 `Cargo.toml` 添加我们对 `futures` 的依赖。

```toml
futures = "0.3"
```

接着实现 `futures::task::ArcWake`

```rust
use futures::task::ArcWake;
use std::sync::Arc;

impl ArcWake for Task {
  fn wake_by_ref(arc_self: &Arc<Self>) {
    arc_self.schedule();
  }
}
```

当我们之前实现的计时器线程调用了 `waker.wake()` 时，该任务将会被发送到 `Channel`，然后我们的调度器的`MiniTokio::run()` 函数会接收到该任务并执行他。

```rust
impl MiniTokio {
    fn run(&self) {
        while let Ok(task) = self.scheduled.recv() {
            task.poll();
        }
    }

    /// Initialize a new mini-tokio instance.
    fn new() -> MiniTokio {
        let (sender, scheduled) = channel::unbounded();

        MiniTokio { scheduled, sender }
    }

    /// Spawn a future onto the mini-tokio instance.
    ///
    /// The given future is wrapped with the `Task` harness and pushed into the
    /// `scheduled` queue. The future will be executed when `run` is called.
    fn spawn<F>(&self, future: F)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        Task::spawn(future, &self.sender);
    }
}

impl Task {
    fn poll(self: Arc<Self>) {
        // Create a waker from the `Task` instance. This
        // uses the `ArcWake` impl from above.
        let waker = task::waker(self.clone());
        let mut cx = Context::from_waker(&waker);

        // No other thread ever tries to lock the future
        let mut future = self.future.try_lock().unwrap();

        // Poll the future
        let _ = future.as_mut().poll(&mut cx);
    }

    // Spawns a new taks with the given future.
    //
    // Initializes a new Task harness containing the given future and pushes it
    // onto `sender`. The receiver half of the channel will get the task and
    // execute it.
    fn spawn<F>(future: F, sender: &channel::Sender<Arc<Task>>)
    where
        F: Future<Output = ()> + Send + 'static,
    {
        let task = Arc::new(Task {
            future: Mutex::new(Box::pin(future)),
            executor: sender.clone(),
        });

        let _ = sender.send(task);
    }

}
```

上面的代码修改了许多的东西，首先 `MiniTokio::run` 的实现方式变更为使用一个循环不停的从 `Channel` 中接收已调度的任务。当任务能被唤醒时将会被放到 `Channel`，这样能够让 `MiniTokio::run` 能够让他继续的推进执行进度。

然后 `MiniTokio::new` 以及 `MiniTokio::spawn` 函数调整为使用 `Channel` 来替换 `VecDeque`。当新的任务被创建时，`Channel` 的 `Sender` 部分传递给任务保存，因此当任务能够继续推进时，能用它来将自己调度给运行时去执行。

`Task::poll` 函数使用 `futures::task::ArcWake` 来便捷的创建 `Waker`，该 `Waker` 会被用来创建任务的上下文，该上下文信息会在之后调用 `poll` 的时候作为参数传递。

## Summary

到现在为止我们看到了一个用来说明 Rust 的异步系统是如何运行的完整例子，Rust 的 `async/await` 是通过 `Trait` 来进行支持实现的，如 `Future` 等，这让第三方的包，如 `Tokio`，能够通过他们来实现具体的细节。

- Rust 的异步操作是需要调用者通过 Poll 去推进的惰性操作
- `Waker` 被传递给 `Future`，`Future` 将使用他来任务进行关联
- 当资源未就绪时会返回 `Poll::Pending`，这时任务的 `Waker` 会被其记录下来
- 当资源就绪时，会使用已记录的 `Waker` 来发出通知
- 执行器接收到通知后会将对应的任务调度执行
- 任务被再次 `Poll`，因为资源此时已就绪所以这次执行能够推进任务的状态

### 一些零散的信息

回顾一下我们早前说的，`Delay` 还有一些问题需要修复，主要的问题在于 Rust 的异步模型允许一个 `Future` 在不同的任务中传递，我们来看看下面的代码

```rust
use futures::future::poll_fn;
use std::future::Future;
use std::pin::Pin;

#[tokio::main]
async fn main() {
  let when = Instant::now() + Duration::from_millis(10);
  let mut delay = Some(Delay { when });
  
  poll_fn(move |cx| {
    let mut delay = delay.take.unwrap();
    let res = Pin::new(&mut delay).poll(cx);
    assert!(res.is_pending());
    tokio::spawn(async move {
      delay.await;
    });
  }).await;
}
```

`poll_fn` 使用一个闭包函数创建了一个 `Future` 实例，上面的代码段创建了一个 `Delay` 的实例，然后对其调用了一次 `poll`，然后又将其发送给了另外一个任务去调用 .await 进行等待。在这个例子中，`Delay::poll` 被调用了多次，并且是使用了 不同的 `Waker` 实例。我们早先的实现没办法处理这种场景，他会导致创建的任务永远处于睡眠状态，因为最后是错误的任务收到了通知。

在实现 `Future` 时一个非常重要的点在于，每次调用 `poll` 传递的 `Waker` 可能并非同一个，`poll` 函数在被调用时需要将自己记录的 `Waker` 替换为新的那个。

为了修复这个问题，我们可以将代码改成下面这样

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::thread;
use std::time::{Duration, Instant};

struct Delay {
    when: Instant,
    waker: Option<Arc<Mutex<Waker>>>,
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // First, if this is the first time the future is called, spawn the
        // timer thread. If the timer thread is already running, ensure the
        // stored `Waker` matches the current task's waker.
        if let Some(waker) = &self.waker {
            let mut waker = waker.lock().unwrap();

            // Check if the stored waker matches the current task's waker.
            // This is necessary as the `Delay` future instance may move to
            // a different task between calls to `poll`. If this happens, the
            // waker contained by the given `Context` will differ and we
            // must update our stored waker to reflect this change.
            if !waker.will_wake(cx.waker()) {
                *waker = cx.waker().clone();
            }
        } else {
            let when = self.when;
            let waker = Arc::new(Mutex::new(cx.waker().clone()));
            self.waker = Some(waker.clone());

            // This is the first time `poll` is called, spawn the timer thread.
            thread::spawn(move || {
                let now = Instant::now();

                if now < when {
                    thread::sleep(when - now);
                }

                // The duration has elapsed. Notify the caller by invoking
                // the waker.
                let waker = waker.lock().unwrap();
                waker.wake_by_ref();
            });
        }

        // Once the waker is stored and the timer thread is started, it is
        // time to check if the delay has completed. This is done by
        // checking the current instant. If the duration has elapsed, then
        // the future has completed and `Poll::Ready` is returned.
        if Instant::now() >= self.when {
            Poll::Ready(())
        } else {
            // The duration has not elapsed, the future has not completed so
            // return `Poll::Pending`.
            //
            // The `Future` trait contract requires that when `Pending` is
            // returned, the future ensures that the given waker is signalled
            // once the future should be polled again. In our case, by
            // returning `Pending` here, we are promising that we will
            // invoke the given waker included in the `Context` argument
            // once the requested duration has elapsed. We ensure this by
            // spawning the timer thread above.
            //
            // If we forget to invoke the waker, the task will hang
            // indefinitely.
            Poll::Pending
        }
    }
}
```

这个改动并不大，主要的想法是，在每次调用 `Future::poll` 时检查参数的 `Waker` 是否与之前保存的一致，如果一致则不需要做任何处理，否则的话则需要将保存的 Waker 替换为新的那个。

### Notify Utility

我们示范了如何手动的使用 `Waker` 来实现 `Delay Future`，`Waker` 是 Rust 的异步机制实现的基础支撑，但一般情况下我们并不需要深入到如此底层的东西，比如说在实现 `Delay` 时可以使用 `async/await` 并借助 `tokio::sync::Notify` 工具来避免复杂的底层信息。该工具提供了基础的任务通知机制，该机制处理了关于 `Waker` 的细节，包括如何保障当前记录的 `Waker` 跟最新的一致。

使用 `Notify` 后我们的 `Delay` 实现看起来成了这样

```rust
use tokio::sync::Notify;
use std::sync::Arc;
use std::time::{Duration, Instant};
use std::thread;

async fn delay(dur: Duration) {
  let when = Instant::now() + dur;
  let notify = Arc::new(Notify::new());
  let notify2 = notify.clone();
  
  thread::spawn(move || {
    let now = Instant::now();
    
    if now < when {
      thread::sleep(when - now);
    }
    
    notify2.notify_one();
  });
  
  notify.notified().await;
}
```