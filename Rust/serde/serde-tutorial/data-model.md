## Serde 数据模型

Serde 数据模型是与数据结构和数据格式进行交互的 API。你可以将其视为 Serde 的类型系统。

在代码中，Serde 数据模型的序列化部分是由 [Serializer](https://docs.serde.rs/serde/trait.Serializer.html) 特征定义的，反序列化部分是由 [Deserializer] 特征定义的。这是将每个 Rust 数据结构映射成 29 种可能类型之一的方式。`Serializer`的每一个方法都对应数据模型的一种类型。

当将一个数据结构序列化成某个格式时，该数据结构的 [Serialize](https://docs.serde.rs/serde/trait.Serialize.html) 实现负责通过调用`Serializer`的一个方法将数据结构映射成 Serde 数据模型，而数据格式的`Serializer`实现负责将 Serde 数据模型映射成预期的输出表示形式。

当将某个格式反序列化成一个数据结构时，该数据结构的 [Deserialie](https://docs.serde.rs/serde/trait.Deserialize.html) 实现通过传递给`Deserializer`一个可以接收到数据模型的不同类型的 [Visitor](https://docs.serde.rs/serde/de/trait.Visitor.html) 实现将数据结构映射成 Serde 数据模型，而数据格式的`Deserializer`实现负责通过调用`Visitor`的一个方法将输入数据映射成 Serde 数据模型。

## 类型

Serde 数据模型是 Rust 类型系统的简化形式。它包含了以下 29 种类型：

- 14 种 原始类型
  - bool
  - i8, i16, i32, i64, i128
  - u8, u16, u32, u64, u128
  - f32, f64
  - char
- 字符串
  - 具有长度且没有空终止符的 UTF
  -8 字节。可能包含 0 个字节。
  - 当序列化时，所有的字符串做同等处理。当反序列化时，这里有三种类型的字符串：暂时的，拥有所有权以及借用的字符串。它们之间的区别在[理解反序列化器的生命周期](./理解反序列化器的生命周期.md)中做了解释，这是 Serde 启用有效的零拷贝反序列化的一种关键方式。
- 字节数组 - [u8]
  - 和字符串类似，在反序列化时，字节数组可以是暂时的，拥有所有权以及借用的。
- option
  - None 或者其他值
- 单元
  - Rust 中的 ()。它代表一个不包含任何值的匿名数据。
- 单元结构体
  - 如：`struct Unit`或者`PhantomData<T>`。它代表了一个被命名的值，但不包含任何数据。
- 单元变量
  - 如：`enum E { A, B }`中的`E::A`和`E::B`。
- 新类型结构体
  - 如：`struct Millimeters(u8)`。
- 新类型变量
  - 如：`enum E { N(u8) }`中的`E::N`。
- 序列
  - 大小可变的值的异质 (heterogeneous) 值的序列，如：`Vec<T>`或`HashSet<T>`。序列化时，在遍历所有数据之前的长度可能已知，也可能未知。反序列化时，通过查看序列化数据来确定长度。请注意，Rust 的同质 (homogeneous) 集合(如：`vec![Value::Bool(true), Value::Char('c')]`)可能会被序列化成异质的 Serde 序列，在这种情况下，包含一个 Serde bool 值及 Serde char 值。
- 元组
  - 静态大小的异质值的序列，在反序列化时无需查看序列化数据即可知道其长度，如： `(u8)`或者`(String, u64, Vec<T>)`或者`[u64; 10]`。
- 元组结构体
  - 一个被命名的元组，如：`struct Rgb(u8, u8, u8)`。
- 元组变量
  - 如：`enum E { T(u8, u8) }`中的`E::T`。
- map
  - 大小可变的异质键值对，如：`BTreeMap<K, V>`。序列化时，在遍历所有的 entry 之前，长度可能已知，也可能未知。反序列化时，通过查看序列化数据来确定长度。
- 结构体
  - 静态大小的异质键值对，其中键是编译时常量字符串，并且在反序列化时就知道，无需查看序列化数据，如：`struct S { r: u8, g: u8, b: u8 }`。
- 结构体变量

  - 如：`enum E { S {r:u8, g:u8, b: u8} }`中的`E::S`。

## 映射到数据模型

对于大多数 Rust 类型，将它们映射到 Serde 数据模型中非常简单。如：Rust 的 bool 类型对应于 Serde 的 bool 类型。Rust 的元组结构体`Rgb(u8,u8,u8)`对应于 Serde 的元组结构体类型。

但是，从根本上来说，没有理由要求这些映射必须简单明了。[Serialize](https://docs.serde.rs/serde/trait.Serialize.html) 和 [Deserialize](https://docs.serde.rs/serde/trait.Deserialize.html) 特征可以在 Rust 类型以及适用于用例的 Serde 数据模型之间执行任何映射。

举个例子，对于 Rust 中的 [std::ffi::OsString](https://doc.rust-lang.org/std/ffi/struct.OsString.html) 类型，它代表一个平台原生的字符串。在 Unix 系统中，它们是任意 (arbitrary) 非零字节，在 Windows 系统中，它们是任意 16 位值。将`OsString`作为以下某一类型映射到 Serde 数据模型中就显得很自然了：
 - 作为一个 Serde 字符串。不幸的是，由于`OsString`不能保证可以使用 UTF-8 表示，它的序列化会显得很脆弱 (brittle)。由于 Serde 字符串允许包含 0 个字节，所以它的反序列化也比较脆弱。
 - 作为一个 Serde 字节数组。它修复了使用字符串的两个问题，但是现在如果我们在 Unix 上将其序列化并在 Windows 上将其反序列化，我们最终会得到一个[错误的字符串](https://www.joelonsoftware.com/2003/10/08/the-absolute-minimum-every-software-developer-absolutely-positively-must-know-about-unicode-and-character-sets-no-excuses/)。

解决办法是，通过将`OsString`视为 Serde **枚举**，将`OsString`的`Serialize`与`Deserialize`实现映射到 Serde 数据模型中。实际上，它的行为就像`OsString`被定义为以下类型，即使这与任何单个平台的定义都不匹配。

```rust
enum OsString {
    Unix(Vec<u8>),
    Windows(Vec<u16>),
    // and other platforms
}
```

映射到 Serde 数据模型的灵活性非常强大。在实现`Serialize`与`Deserialize`时，请注意类型更广阔的上下文，这可能使最本能的 (instinctive) 的映射不是最佳选择。