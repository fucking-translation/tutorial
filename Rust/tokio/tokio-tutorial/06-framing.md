# Framing

[原文](https://tokio.rs/)

</br>


我们现在来将我们上一章学到的 I/O 相关的知识用来实现 Mini-Redis 的帧模块。帧模块将字节流进行处理并转换为一个个的数据帧，每个帧表示了两端之间数据传输的命令单元。Redis 协议的帧如下所示

```rust
use bytes::Bytes;

enum Frame {
  Simple(String),
  Error(String),
  Integer(u64),
  Bulk(Bytes),
  Null,
  Array<Vec<Frame>),
}
```

帧的结构并不能代表任何语义，因为具体的命令解析跟实现是在更上一层做的，对于 HTTP 来说，一个帧看起来像这样

```rust
enum HttpFrame {
  RequestHead: {
    method: Method,
    uri: Uri,
    version: Version,
    headers: HeaderMap,
  },
  ResponseHead {
    status: StatusCode,
    version: Version,
    headers: HeaderMap,
  },
  BodyChunk {
    chunk: Bytes
  }
}
```

为了实现 Mini-Redis 的帧，我们会实现一个 `Connection` 结构，他对 `TcpStream` 以及 `mini_redis::Frame` 的读取跟写入进行了封装。

```rust
use mini_redis::{Frame, Result};
use tokio::net::TcpStream;

struct Connection {
    stream: TcpStream,
    // ... other fields here
}

impl Connection {
    /// Read a frame from the connection
    ///
    /// Returns `None` if EOF is reached
    pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
        Ok(None)
    }

    // Write a frame to the connection
    pub async fn write_frame(&mut self, frame: &Frame) -> Result<()> {
        Ok(())
    }
}
```

对于具体的 Redis 协议可以参考[这里](https://redis.io/topics/protocol))，完整的 `Connection` 代码可以从 这里 找到。

## Buffered Reads

`read_frame` 函数会执行到从 `stream` 读取到一个完整的帧之后才返回，因为一个单独的 `TcpStream::read` 调用返回指定数量的数据，所以他可能返回一个完整的帧、部分的帧或者是多个帧，如果读取到了部分的帧，数据将被缓存起来，然后尝试继续从套接字中继续读取；如果收到了多个帧，第一个帧会马上返回并且剩余的数据会缓存起来，直到下一次的 `read_frame` 调用。

为了实现 `Connection` 我们需要一个读取缓存的字段，数据从套接字中读取到缓存中，当数据能够组成一个帧时，该帧对应的数据会从缓存中删除。

我们会使用 `BytesMut` 作为缓存类型，这是一个跟 `Bytes` 一样但允许修改的类型。

```rust
/// Read a frame from the connection
///
/// Returns `None` if EOF is reached
pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
  loop {
    // Attempt to parse a frame from the buffered data. If
    // enough data has been buffered, the frame is
    // returned.
    if let Some(frame) = self.parse_frame()? {
      return Ok(Some(frame));
    }

    // There is not enough buffered data to read a frame.
    // Attempt to read more data from the socket.
    //
    // On success, the number of bytes is returned. `0`
    // indicates `end of stream'.
    if 0 == self.stream.read_buf(&mut self.buffer).await? {
      return if self.buffer.is_empty() {
        Ok(None)
      } else {
        Err("Connection reset by peer".into())
      }
    }
  }
  Ok(None)
}
```

接着来逐步分解代码，`read_frame` 函数中的处理都处于 `loop` 循环内，第一步会尝试调用 `self.parse_frame()` 去从缓存 `self.buffer` 中解析出 Redis 的帧，如果缓存中的数据足够组成一个帧，则马上返回解析出来的帧。否则的话则尝试从套接字中读取更多的数据到缓存中，然后在下一个循环中继续尝试调用 `parse_frame`，如果这时数据足够了，则解析帧的操作就能够成功。

当从数据流中读取数据时，返回值 `0` 用来确认是否已经没有数据可以继续从套接字中读取了，如果这时缓存中仍有数据，说明我们读取到了部分的帧并且连接已经被意外中断了，当出现这种情况时 `read_frame` 会返回一个 `Err` 错误。

## The Buf Trait

当从数据流中进行读取时会调用 `read_buf`，当前版本的读取函数是由 `bytes` 包的 `BufMut` 类型实现的。

现在来考虑一下怎么同时在循环中通过 `read()` 来实现一样的功能， 先用 `Vec<u8>` 来替换掉 `BytesMut`。

```rust
use tokio::net::TcpStream;

pub struct Connection {
  stream: TcpStream,
  buffer: Vec<u8>,
  cursor: usize,
}

impl Connection {
  pub fn new(stream: TcpStream) -> Connection {
    Connection {
      stream,
      buffer: vec![0; 4096],
      cursor: 0,
    }
  }
}
```

然后是 `read_frame` 函数的实现

```rust
/// Read a frame from the connection
///
/// Returns `None` if EOF is reached
pub async fn read_frame(&mut self) -> Result<Option<Frame>> {
  loop {
    // Attempt to parse a frame from the buffered data. If
    // enough data has been buffered, the frame is
    // returned.
    if let Some(frame) = self.parse_frame()? {
      return Ok(Some(frame));
    }

    // Ensure the buffer has capacity
    if self.buffer.len() == self.cursor {
      // Grow the buffer
      self.buffer.resize(self.cursor * 2, 0);
    }

    // Read into the buffer, tracking the number
    // of bytes read
    let n = self.stream.read(&mut self.buffer[self.cursor..]).await?;

    if 0 == n {
      return if self.cursor == 0 {
        Ok(None)
      } else {
        Err("Connection reset by peer".into())
      };
    } else {
      self.cursor += n;
    }
  }
}
```

使用字节数组跟 `read` 来实现功能，我们需要自己管理游标来跟踪当前已经缓存的数据量，确保只把缓存中的未使用区域传递给 `read()`，否则的话会导致缓存被覆盖。如果缓存被填满了，我们需要扩充缓存的大小用来填充更多的数据。在 `parse_frame()` 中，我们需要用到缓存中 `self.buffer[..self.cursor]` 的区域。

因为结对使用字节数组与游标是很常用的方式，`bytes` 包提供了对他们进行了抽象提取，最后以 `Buf Trait` 提供了数据读取及 `ByteMut Trait` 提供了数据写入的能力。当传递了一个 `T: BufMut` 类型给 `read_buf()` 函数时，该类型内部的游标会由 `read_buf` 更新。 因此在我们之前实现的 `read_frame` 版本中不需要自己来管理游标信息。

另外，在使用 `Vec<u8>` 时我们还需要对其进行初始化，`vec![0; 4096]` 分配了 4096 个字节的数组并将每个字节使用了 0 来进行填充，当扩充缓存的大小时，扩充部分的数据也需要使用 0 来进行填充。这个初始化的过程并不是没有代价的，但在使用 `BytesMut` 跟 `BufMut` 时，缓存的容量是未初始化的， BytesMut 抽象能够防止我们读取未初始化的内存，因此得以让我们避免了初始化这一步。

## Parsing

现在让我们来看一下 `parse_frame()` 函数，解析的操作分成了两步来实现

1. 确认缓存中包含了完整的帧并找到该帧所在最后一个字节的位置
2. 解析该帧

`mini-redis` 包为我们要做的两步都提供了对应的函数

1. `Frame::check`
2. `Frame::parse`

我们会继续使用 `Buf` 抽象来简化操作，因此会将一个 `Buf` 传递给 `Frame::check` ，当 `check` 函数迭代传递给它的缓存时，他内部的游标会被往前移动，当 `check` 函数返回时，缓存内部的游标会指向该帧的最后一个位置。

然后我们会使用 `std::io::Cursor<&[u8]>` 得到一个 `Buf` 类型。

```rust
use mini_redis::{Frame, Result};
use mini_redis::frame::Error::Incomplete;
use bytes::Buf;
use std::io::Cursor;

fn parse_frame(&mut self) -> Result<Option<Frame>> {
  // Create the `T: Buf` type
  let mut buf = Cursor::new(&self.buffer[..]);

  // Check whether a full frame is available
  match Frame::check(&mut buf) {
    Ok(_) => {
      // Get the byte length of the frame
      let len = buf.position() as usize;

      // Reset the internal cursor for the
      // call to `parse`
      buf.set_position(0);

      // Parse the frame
      let frame = Frame::parse(&mut buf)?;

      // Discard the frame from the buffer
      self.buffer.advance(len);

      Ok(Some(frame))
    }
    Err(Incomplete) => Ok(None),
    Err(e) => Err(e.into()),
  }
}
```

完整的 `Frame::check` 函数可以在[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/frame.rs#L63-L100)找到，在这里我们不会讨论他的具体实现。

这里要注意的是 `Buf` 被看为了一个 "字节迭代器" 风格的接口来使用，这些接口会获取其中的数据并往前移动其游标。比如解析帧时，第一个字节会被用来作为帧的类型进行检查，最后内部使用 `Buf::get_u8` 来获取数据及移动游标信息。

`Buf Trait` 还提供了其他许多有用的接口，具体的可以查阅其[接口文档](https://docs.rs/bytes/1/bytes/buf/trait.Buf.html)。

## Buffered Writes

另一部分处理帧的函数是 `write_frame(frame)`，这个函数将整个帧的数据写到套接字中，为了最小化 `write` 系统调用，写入的数据也会被缓存起来，写入的帧被编码后会缓存起来直到最后一起写入套接字，然而，不像 `read_frame` 函数，整个帧并不总是能够在写入套接字前缓存起来。

考虑一个批量的数据流帧，他被写入的数据是 `Frame::Bulk(Bytes)` 类型。写入的数据类型是一个帧的头部，该头部以 `$` 字符开头，后接剩余的字节数据长度。该帧的主要部分是他的内容 `Bytes` 的值，如果数据太大，复制他们到中间的缓存区域将会是一个比较大的开销。

为了实现缓存的写入，我们会使用 `BufWriter` 类型，该类型有一个实现了 `T: AsyncWrite` 的类型进行初始化，同时他自身也实现了 `AsyncWrite`。当他的 `write` 函数被调用时，该写入操作并不会直接的传递给内部的写入器，而是会写到缓存中。在缓存被填满时，缓存的内容才会被写入到其内部的写入器，然后缓存的内容会被清空，这里面还有一些允许绕过缓存区的优化。

我们不会在教程中尝试完整的实现 `write_frame()`，完整的代码可以从[这里](https://github.com/tokio-rs/mini-redis/blob/tutorial/src/connection.rs#L159-L184)获取到。

首先， `Connection` 类型被更新为：

```rust
use tokio::io::BufWriter;
use tokio::net::TcpStream;
use bytes::BytesMut;

struct Connection {
    stream: BufWriter<TcpStream>,
    buffer: BytesMut, // ... other fields here
}

impl Connection {
    pub fn new(stream: TcpStream) -> Connection {
        Connection {
            stream: BufWriter::new(stream),
            buffer: BytesMut::with_capacity(4096),
        }
    }
}
```

然后 `write_frame()` 被实现为：

```rust
use tokio::io::{self, AsyncWriteExt};
use mini_redis::Frame;

async fn write_frame(&mut self, frame: &Frame)
-> io::Result<()>
{
  match frame {
    Frame::Simple(val) => {
      self.stream.write_u8(b'+').await?;
      self.stream.write_all(val.as_bytes()).await?;
      self.stream.write_all(b"\r\n").await?;
    }
    Frame::Error(val) => {
      self.stream.write_u8(b'-').await?;
      self.stream.write_all(val.as_bytes()).await?;
      self.stream.write_all(b"\r\n").await?;
    }
    Frame::Integer(val) => {
      self.stream.write_u8(b':').await?;
      self.write_decimal(*val).await?;
    }
    Frame::Null => {
      self.stream.write_all(b"$-1\r\n").await?;
    }
    Frame::Bulk(val) => {
      let len = val.len();

      self.stream.write_u8(b'$').await?;
      self.write_decimal(len as u64).await?;
      self.stream.write_all(val).await?;
      self.stream.write_all(b"\r\n").await?;
    }
    Frame::Array(_val) => unimplemented!(),
  }

  self.stream.flush().await;

  Ok(())
}
```

该函数使用了 `AsyncWriteExt` 提供的功能，他同样也能使用 `TcpStream` 来调用，但这样的话就没办法在写入时使用到中间的缓存区域了。

- `write_u8` 写入一个单独的字节
- `write_all` 写入所有的数据
- `write_decimal` 则是有 mini-redis 所实现

该函数最后还要调用 `self.stream.flush().await`，因为对 `BufWriter` 会将写入的数据缓存起来，`write` 操作并不能保证数据会写入到套接字，在返回前，我们希望帧的所有数据都会被写入到套接字中，所以通过 `flush` 函数实现将所有的数据写入到套接字。

另一个选择是不在 `write_frame()` 中调用 `flush()`，而是为 `Connection` 类型提供一个 `flush()` 函数，这将允许调用者将多个较小的帧缓存起来，最后使用一个 `write` 系统调用一次性将他们写入套接字。这个实现会让 `Connection` 变的复杂一些，简单是 Mini-Redis 的目标之一，所以我们选在直接将 `flush().await` 的调用直接写到了 `write_frame()` 里。