# 自定义序列化器

Serde 的[派生宏](https://serde.rs/derive.html)通过`#[derive(Serialize, Deserialize)]`为结构体和枚举提供了默认的序列化行为，并且它可以通过使用[属性](https://serde.rs/attributes.html)来自定义一些扩展。对一些不常见的需求，Serde 允许通过手动实现 [Serialize](https://docs.serde.rs/serde/ser/trait.Serialize.html) 和 [Deserialize](https://docs.serde.rs/serde/de/trait.Deserialize.html) 来为你的类型自定义序列化行为。

这两个特征都只有一个方法：

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}

pub trait Deserialize<'de>: Sized {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>;
}
```

这些方法是序列化格式的通用方法，由 [Serializer](https://docs.serde.rs/serde/ser/trait.Serializer.html) 和 [Deserializer](https://docs.serde.rs/serde/de/trait.Deserializer.html) 特征表示。举个例子：JSON 有一种 Serializer 类型，而 Bincode 有另一种类型。

* [实现 `Serializer`](https://serde.rs/impl-serialize.html)
* [实现 `Deserializer`](https://serde.rs/impl-deserialize.html)