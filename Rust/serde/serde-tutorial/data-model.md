## Serde 数据模型

Serde 数据模型是与数据结构和数据格式进行交互的 API。你可以将其视为 Serde 的类型系统。

在代码中，Serde 数据模型的序列化部分是由 [Serializer](https://docs.serde.rs/serde/trait.Serializer.html) 特征定义的，反序列化部分是由 [Deserializer] 特征定义的。这是将每个 Rust 数据结构映射成 29 种可能类型之一的方式。`Serializer`的每一个方法都对应数据模型的一种类型。

当将一个数据结构序列化成某个格式时，该数据结构的 [Serialize](https://docs.serde.rs/serde/trait.Serialize.html) 实现负责通过调用`Serializer`的一个方法将数据结构映射成 Serde 数据模型，而数据格式的`Serializer`实现负责将 Serde 数据模型映射成预期的输出表示形式。