## 错误处理

在序列化过程中，[Serialize](https://docs.serde.rs/serde/trait.Serialize.html) 特征将 Rust 数据结构映射成 Serde [数据模型](./data-model.md)，[Serializer](https://docs.serde.rs/serde/ser/trait.Serializer.html) 特征将数据模型映射成输出格式。在反序列化过程中，[Derserializer](https://docs.serde.rs/serde/trait.Deserializer.html) 将输入数据映射成 Serde 数据模型，[Deserialize](https://docs.serde.rs/serde/trait.Deserialize.html) 和 [Visitor](https://docs.serde.rs/serde/de/trait.Visitor.html) 特征将数据模型映射成 Rust 数据结构。这些所有步骤都可能失败。

- `Serialize`可能失败，例如：当`Mutex<T>`正在被序列化，mutex 发生中毒时。
- `Serializer`可能失败，例如：Serde 数据模型允许映射非字符串的键而 JSON 不行。
- `Deserializer`可能失败，尤其是输入数据在语法上无效时。
- `Deserialize`可能失败，通常是因为对于反序列化的值来说，输入是错误的类型。

在 Serde 中，`Serializer`和`Deserializer`的工作方式与其他 Rust 库一样。这个 crate 定义了错误类型，公共函数返回一个带错误类型的 Result，并且这里有多种可能错误模式的变体。

处理`Serialize`和`Deserialize`处理错误，是围绕 [ser::Error](https://docs.serde.rs/serde/ser/trait.Error.html) 和 [de::Error](https://docs.serde.rs/serde/de/trait.Error.html) 进行构建的。这些特征允许数据格式公开其错误类型的构造函数，以供数据结构在各种情况下使用。

```rust
use std;
use std::fmt::{self, Display};

use serde::{de, ser};

pub type Result<T> = std::result::Result<T, Error>;

// This is a bare-bones implementation. A real library would provide additional
// information in its error type, for example the line and column at which the
// error occurred, the byte offset into the input, or the current key being
// processed.
#[derive(Clone, Debug, PartialEq)]
pub enum Error {
    // One or more variants that can be created by data structures through the
    // `ser::Error` and `de::Error` traits. For example the Serialize impl for
    // Mutex<T> might return an error because the mutex is poisoned, or the
    // Deserialize impl for a struct may return an error because a required
    // field is missing.
    Message(String),

    // Zero or more variants that can be created directly by the Serializer and
    // Deserializer without going through `ser::Error` and `de::Error`. These
    // are specific to the format, in this case JSON.
    Eof,
    Syntax,
    ExpectedBoolean,
    ExpectedInteger,
    ExpectedString,
    ExpectedNull,
    ExpectedArray,
    ExpectedArrayComma,
    ExpectedArrayEnd,
    ExpectedMap,
    ExpectedMapColon,
    ExpectedMapComma,
    ExpectedMapEnd,
    ExpectedEnum,
    TrailingCharacters,
}

impl ser::Error for Error {
    fn custom<T: Display>(msg: T) -> Self {
        Error::Message(msg.to_string())
    }
}

impl de::Error for Error {
    fn custom<T: Display>(msg: T) -> Self {
        Error::Message(msg.to_string())
    }
}

impl Display for Error {
    fn fmt(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        match self {
            Error::Message(msg) => formatter.write_str(msg),
            Error::Eof => formatter.write_str("unexpected end of input"),
            /* and so forth */
        }
    }
}

impl std::error::Error for Error {}
```