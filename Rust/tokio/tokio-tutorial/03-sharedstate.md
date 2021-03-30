# Shared State

[原文](https://tokio.rs/)

</br>

现在我们有一个可以运行的键值对服务端了，但是还有一个明显的瑕疵：状态不能跨多个连接共享，在这篇文章中我们来解决这个问题。

## Strategies
在 Tokio 中有几种不同的方式来实现共享状态。

1. 通过 Mutex 保护共享的状态
2. 创建一个新的任务管理状态并通过消息传递来处理状态

通常你会希望使用第一种方式来处理简单的数据，第二种方式一般用来处理那些需要异步处理的 **I/O** 设施。 在当前章节，共享的状态是一个提供了`insert`跟`get`操作的`HashMap`，这两个操作都不是异步的，所以我们选择使用`Mutex`。

## Add`bytes`dependency

Mini-Redis 包使用了包`bytes`中的`Bytes`类型替代了`Vec<u8>`。`Bytes`的目标是为网络编程提供健壮的字节数组结构，他在 `Vec<u8>`之上提供的最大的特性就是浅拷贝。换一种说法就是，调用`Bytes`的`clone`函数并不会复制底层的任何数据，取而代之的是，`Bytes`使用了引用计数的方式来处理底层的数据，它类似于`Arc<Vec<u8>>`但提供了一些其他的能力。

为了添加对他的依赖，我们需要在`Cargo.toml`中的`[denpendencies]`中添加下面的信息

```toml
bytes = "1"
```

## Initialize the HashMap

`HashMap`将被用来在不同的线程间进行共享，为了实现这个目的我们使用`Arc<Mutex<_>>`进行包装。

第一步，为了后续方便使用，先使用`use`引入下面的类型；

```rust
use bytes::Bytes;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

type db = Arc<Mutex<HashMap<String, Bytes>>>;
```

接着更新`main`函数来初始化`HashMap`，然后将它作为一个`Arc`的句柄传递给`process`函数。使用`Arc`将允许我们将`HashMap`当成一个引用传递多个任务，而这些任务可能会运行在不同的线程上。在 Tokio 中**Handle**一般表示为用来访问一个共享状态的值的引用。

```rust
use tokio::net::TcpListener;
use std::collections::HashMap;
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() {
    let listener = TcpListener::bind("127.0.0.1:6379").await.unwrap();

    println!("Listening");

    let db = Arc::new(Mutex::new(HashMap::new()));

    loop {
        let (socket, _) = listener.accept().await.unwrap();
        // Clone the handle to the hash map.
        let db = db.clone();

        println!("Accepted");
        tokio::spawn(async move {
            process(socket, db).await;
        });
    }
}
```

## On using std::sync::Mutex

注意，使用`std::sync::Mutex`而不是`tokio::sync::Mutex`来保护`HashMap`。一个常见的误用就是在异步的代码中使用 `tokio::sync::Mutex`，异步的`Mutex`是用来保护多个`.await`之间的调用的。

同步的`Mutex`在尝试获取锁时会堵塞当前线程，意味着他同时也会堵塞其他的任务。然而，切换为`tokio::sync::Mutex`通常不会带来什么帮助，因为异步的`Mutext`在内部也是使用同步的`Mutext`。

作为一个指导规则，在异步的代码中使用同步的`Mutex`不会有什么问题，只要操作评率保持较低，并且持有锁的操作不跨越多个`.await`。 除此之外，使用`parking_log::Mutex`是个更快的替换`std::sync::Mutex`的方案。

## Update process()

`process`函数不再初始化`HashMap`，而是通过参数获取一个`HashMap`的句柄，并且在使用之前要对其进行加锁。

```rust
use tokio::net::TcpStream;
use mini_redis::{Connection, Frame};

async fn process(socket: TcpStream, db: Db) {
    // The `Connection` lets us read/write redis **frame** instead of
    // byte streams. The `Connection` type is defined by mini-redis

    use mini_redis::Command::{self, Get, Set};
    use std::collections::HashMap;

    // Connection, provided by `mini-redis`, handles parsing frames from
    // the socket
    let mut connection = Connection::new(socket);

    while let Some(frame) = connection.read_frame().await.unwrap() {
        let response = match Command::from_frame(frame).unwrap() {
            Set(cmd) => {
                let mut db = db.lock().unwrap();
                db.insert(cmd.key().to_string(), cmd.value().clone());
                Frame::Simple("OK".to_string())
            }
            Get(cmd) => {
                let db = db.lock().unwrap();
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
}
```

## Tasks, threads, and contention

在竞争比较小的情况中使用堵塞的`Mutex`来保护一个短小的临界区是一个可以接收的策略，当获取锁产生竞争时，执行当前任务的线程会因为等待这个`Mutex`而被堵塞住，而且他并不是只堵塞当前任务，而是堵塞所有被调度到这个线程的任务。

在默认的情况下， Tokio 的`Runtime`使用基于多线程的调度器，任务可能会被调度到`Runtime`所管理的任意一个线程中。如果大量调度中的任务都需要访问同一个`Mutex`，那他将会成为一个瓶颈。换种说法，如果使用了`Runtime`的`current_thread`模式 ，那这`Mutex`永远都不可能被获取到。

> `current_thread runtime flavor`是一个轻量的、单线程`Runtime`。在只需要创建少量任务并且处理少量套接字的情况下，他是一个不错的选择。比如为客户端的异步函数提供一个同步接口的桥梁时，他就能工作的很好。

如果同步`Mutex`的竞争成为了程序的瓶颈，最好的修复方式是将它替换为 Tokio 的`Mutext`, 或者是下面的几个方式：

- 使用单独的任务通过消息传递来管理状态信息
- 分区 Mutex
- 重构代码避免使用 Mutex

在我们的例子中，因为每个`Key`都是独立的，使用共享的`Mutex`会是一个较好的方式，为了实现这个目标，我们将单个`Mutex<HashMap<_, _>>`替换为`N`个不同的实例。


```rust
type ShardedDb = Arc<Vec<Mutex<HashMap<String, Vec<u8>>>>>;
```

所以获取某个`Key`对应的存储则变为两步操作，第一步使用`Key`来确认使用哪个共享的元素，第二步才是获取该元素中所使用的`HashMap`。

```rust
let shard = db[hash(key) % db.len()].lock().unwrap();
shard.insert(key, value);
```

有一个`dashmap`包提供已经实现好的分区`HashMap`。

## Holding a MutexGuard across an .await

你可能会写出类似下面的代码

```rust
use std::sync::Mutex;

async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
  let mut lock = mutex.lock().unwrap();
  *lock += 1;
  
  do_somthing_async().await;
} // lock goes out of scope here
```

当你尝试用这个代码来创建任务时，会得到如下的错误信息

```console
error: future cannot be sent between threads safely
   --> src/lib.rs:13:5
    |
13  |     tokio::spawn(async move {
    |     ^^^^^^^^^^^^ future created by async block is not `Send`
    |
   ::: /playground/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-0.2.21/src/task/spawn.rs:127:21
    |
127 |         T: Future + Send + 'static,
    |                     ---- required by this bound in `tokio::task::spawn::spawn`
    |
    = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/lib.rs:7:5
    |
4   |     let mut lock = mutex.lock().unwrap();
    |         -------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
...
7   |     do_something_async().await;
    |     ^^^^^^^^^^^^^^^^^^^^^^^^^^ await occurs here, with `mut lock` maybe used later
8   | }
    | - `mut lock` is later dropped here
```

这是因为`std::sync::MutexGuard`这个类型并非`Send`的。这意味着你不能将`Mutex`锁传递给其他的线程，这个错误会出现则是因为 Tokio 会在每次`.await`时在线程间移动这个任务。为了避免这个问题，应该重构代码，让`Mutex`的锁在调用`.await`前销毁。

```rust
// This works!
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
  {
    let mut lock = mutex.lock().unwrap();
    *lock += 1;
  }
  do_something_async().await;
}
```

要注意的是，下面的方式并不能正常运行

```rust
use std::sync::Mutex;

// This fails too.
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
    let mut lock = mutex.lock().unwrap();
    *lock += 1;
    drop(lock);

    do_something_async().await;
}
```

这是因为编译器当前只会使用当前作用域的信息来判断一个`Future`是否满足`Send`。在将来的某个时候编译器可能会升级来实现分析`drop`操作，但现在你必须自己明确的指定作用域。

关于这个错误的讨论也可以在[Send bound section from the spawning chapter](https://tokio.rs/tokio/tutorial/spawning#send-bound) 中找到。

你不该使用某种不要求`Send`的方式来创建任务，去尝试避免这个问题。因为 Tokio 在执行`.await`时将持有着锁的任务暂定，然后其他的任务会被调度到当前的线程，如果这个任务也尝试去获取这个锁，就会导致这个任务因为获取不到锁被堵塞，同时前一个持有锁的任务可能会因为没有线程可用而无法重新启用，所以无法释放他持有的锁，从而造成死锁。

我们会在后续继续讨论如果解决这个问题。

### Restructure you code to not hold the lock across an .await

我们已经在上面看过一个解决问题的代码示例了，在这里我们提供一种更健壮的方式来实现。比如我们可以将`Mutex`包装到一个结构体里面，并且只会在同步的函数中对其进行加锁。

```rust
use std::sync::Mutex;

struct CanIncrement {
  mutex: Mutex<i32>,
}

impl CanIncrement {
  // This function is not marked async
  fn increment(&self) {
    let mut lock = self.mutex.lock().unwrap();
    *lock += 1;
  }
}

async fn increment_and_do_stuff(can_incr: &CanIncrement) {
  ca_incr.increment();
  do_something_async().await;
}
```

这种方式保证了不会触发`Send`错误，因为`MutexGuard`并没有出现在异步函数中。

### Spawn a task to manage the state and use message passing to operate on it

我们之前提到的第二种方式通常使用在共享的 **IO** 资源的情况，在下一章会详细介绍。

### Use Tokio's asynchronous mutex

也可以是用 Tokio 提供的`tokio::sync::Mutex`类型，他主要的特点是允许只有锁跨越多个`.await`调用。但同时，异步的`Mutex`也需要花费比普通`Mutex`更多的代价，所以更多是使用另外的两个方式。

```rust
use tokio::sync::Mutex; // Note! This uses the Tokio mutex

// This compiles!
// (but restructuring the code would be better in this case)
async fn increment_and_do_stuff(mutex: &Mutex<i32>) {
  let mut lock = mutex.lock().await;
  *lock += 1;
  
  do_something_async().await;
} // lock goes out of scope here
```