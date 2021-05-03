# 实现 Deserialize

[Deserialize](https://docs.serde.rs/serde/de/trait.Deserialize.html) 特征看起来像这样：

```rust
pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

该方法的工作是通过为 [Deserializer](https://docs.serde.rs/serde/trait.Deserializer.html) 提供一个由反序列化器驱动以构造一个你的类型实例的 [Visitor](https://docs.serde.rs/serde/de/trait.Visitor.html)，可以将类型映射到 [Serde 数据模型](https://serde.rs/data-model.html)中。

在大多数情况下，Serde 的 [derive](./derive.md) 可以为定义在你的 crate 中的结构体和枚举生成一个适当的`Deserialize`实现。如果 derive 不支持你的类型生成对应的实现，则你可以通过手动实现`Deserialize`特征为该类型自定义反序列化行为。为类型实现`Deserialize`比实现`Serialize`要复杂的多。

`Deserializer`特征支持两种序列化方式：

1. `deserialize_any`方法。自描述的数据格式 (如：JSON) 可以查看序列化数据并判断它代表什么。举个例子，JSON 反序列化器看到一个左半开的花括号 (`{`) 就知道它是一个 map。如果数据格式支持`Deserializer::deserialize_any`，它将使用在输入中看到的任何类型来驱动 Visitor。当反序列化`serde_json::Value`(它是一个枚举，可以代表任何 JSON 文档)时，JSON 使用的就是这种方式。不用知道它在 JSON 文档中是什么，我们可以通过`Deserializer::deserialize_any`将其序列化成`serde_json::Value`。
2. 其他不同的`deserialize_*`方法。无法自描述的格式(如：bincode)需要被告知输入中有什么以便对其反序列化。`deserialize_*`方法是对反序列化器的提示，让它知道如何解释输入的下一个片段。无法自描述的格式不能像`serde_json::Value`一样依赖`Deserializer::deserialize_any`进行反序列化。

当实现`Deserialize`，你应该避免依赖`Deserializer::deserialize_any`，除非你需要被告知输入中有哪些类型。要知道，依赖`Deserializer::deserialize_any`意味着你的数据类型只能从自描述格式进行反序列化，从而排除了 bincode 等其他数据格式。

## Visitor 特征

[Visitor](https://docs.serde.rs/serde/de/trait.Visitor.html) 通过`Deserialize`实现进行实例化，并传递给一个`Deserializer`。该`Deserializer`在`Visitor`上调用一个方法，以构造想要的类型。

这是一个能够从多种类型中反序列化出原始`i32`类型的 Visitor。

```rust
use std::fmt;

use serde::de::{self, Visitor};

struct I32Visitor;

impl<'de> Visitor<'de> for I32Visitor {
    type Value = i32;

    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
        formatter.write_str("an integer between -2^31 and 2^31")
    }

    fn visit_i8<E>(self, value: i8) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        Ok(i32::from(value))
    }

    fn visit_i32<E>(self, value: i32) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        Ok(value)
    }

    fn visit_i64<E>(self, value: i64) -> Result<Self::Value, E>
    where
        E: de::Error,
    {
        use std::i32;
        if value >= i64::from(i32::MIN) && value <= i64::from(i32::MAX) {
            Ok(value as i32)
        } else {
            Err(E::custom(format!("i32 out of range: {}", value)))
        }
    }

    // Similar for other methods:
    //   - visit_i16
    //   - visit_u8
    //   - visit_u16
    //   - visit_u32
    //   - visit_u64
}
```

`Visitor`特征有很多`I32Visitor`尚未实现的方法。让它们保持未实现的状态意味着当它们被调用时会返回[类型错误](https://docs.serde.rs/serde/de/trait.Error.html#method.invalid_type)。举个例子，`I32Visitor`没有实现`Visitor::visit_map`，因此当输入包含 map 时尝试反序列化 i32 会产生类型错误。

## 驱动一个 Visitor

通过将`Visitor`传递给指定的`Deserializer`来反序列化一个值。`Deserializer`将通过输入的数据来调用`Visitor`中的一个方法，来“驱动”该`Visitor`。

```rust
impl<'de> Deserialize<'de> for i32 {
    fn deserialize<D>(deserializer: D) -> Result<i32, D::Error>
    where
        D: Deserializer<'de>,
    {
        deserializer.deserialize_i32(I32Visitor)
    }
}
```

请注意，`Deserializer`不一定会带有类型提示，因此对`deserialize_i32`的调用不一定意味着`Deserializer`会调用`I32Visitor::visit_i32`。举个例子，JSON 对所有带符号的整数类型都一视同仁。JSON 的`Deserializer`对任何带符号的整数都会调用`visit_i64`，对任何不带符号的整数调用`visit_u64`，即使提示它们都是不同的类型。

## 其他示例

- [反序列化 map](https://serde.rs/deserialize-map.html)
- [反序列化结构体](https://serde.rs/deserialize-struct.html)