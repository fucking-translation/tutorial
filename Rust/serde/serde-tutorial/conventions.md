## 约定

通过约定，Serde 数据格式的 crate 在根模块提供了以下特性或在根模块的重新导出。

- 序列化和反序列化的通用错误类型
- 与`std::result::Result<T, Error>`等价的结果定义
- 实现了`serde::Serializer`的序列化器
- 实现了`serde::Deserializer`的反序列化器
- 一个或多个`to_abc`函数取决于要支持的序列化格式的类型。举个例子，`to_string`函数返回一个`String`，`to_bytes`函数返回一个`Vec<u8>`，`to_writer`函数返回一个 [io::Write](https://doc.rust-lang.org/std/io/trait.Write.html)
- 一个或多个`to_xyz`函数取决于要支持的反序列化格式的类型。举个例子，`from_str`函数传入一个`&str`，`from_bytes`函数传入一个`&[u8]`，`from_reader`函数传入一个 [io::Read](https://doc.rust-lang.org/std/io/trait.Read.html)

此外，除了 Serializer 和 Deserializer 之外，提供特定序列化或反序列化的 API 格式应暴露在顶级 ser 和 de 模块下。举个例子，serde_json 提供了可插拔的 pretty-printer 特征`serde_json::ser::Formatter`。

一个基础的数据格式像这样开始。其他三个模块将在之后的页面讨论更多的细节。

```rust
// src/lib.rs
mod de;
mod error;
mod ser;

pub use de::{from_str, Deserializer};
pub use error::{Error, Result};
pub use ser::{to_string, Serializer};
```