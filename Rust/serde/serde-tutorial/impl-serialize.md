# 实现 Serialize

[Serialize](https://docs.serde.rs/serde/ser/trait.Serialize.html) 特征看起来像这样：

```rust
pub trait Serialize {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer;
}
```

这个方法的功能是传入你的类型 (`&self`) 并通过调用给定的 [Serializer](https://docs.serde.rs/serde/ser/trait.Serializer.html) 其中的一个方法将其映射到 [Serde 数据模型](https://serde.rs/data-model.html)。


在大多数场景中，Serde 的[派生](https://serde.rs/derive.html)都可以为你的 crate 中的结构体或枚举生成合适的`Serialize`实现。当此派生不支持某个类型时，你需要为其自定义序列化行为，你可以自己实现`Serialize`。

## 实例化原语

作为一个最简单的例子，这里有一个`i32`原始类型的内置`Serialize`实现。

```rust
impl Serialize for i32 {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        serializer.serialize_i32(*self)
    }
}
```

Serde 为 Rust 的所有[原始类型](https://doc.rust-lang.org/book/primitive-types.html)都提供了这样的实现，因此你不需要再为它们手动实现了，但是如果你有一个类型需要以其序列化的形式表示为原始类型，`serialize_i32`和其他类似的方法将会很有用。举个例子，你可以[将一个类 C 的枚举序列化成一个原始数字](https://serde.rs/enum-number.html)。

## 序列化序列或 map

复合类型遵循初始化，元素，结束三个步骤。

```rust
use serde::ser::{Serialize, Serializer, SerializeSeq, SerializeMap};

impl<T> Serialize for Vec<T>
where
    T: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut seq = serializer.serialize_seq(Some(self.len()))?;
        for e in self {
            seq.serialize_element(e)?;
        }
        seq.end()
    }
}

impl<K, V> Serialize for MyMap<K, V>
where
    K: Serialize,
    V: Serialize,
{
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        let mut map = serializer.serialize_map(Some(self.len()))?;
        for (k, v) in self {
            map.serialize_entry(k, v)?;
        }
        map.end()
    }
}
```

## 序列化元组

`serialize_tuple`方法与`serialize_seq`非常像。唯一的区别就是`serialize_tuple`不需要对元组的长度进行序列化，因为它可以在反序列化期间获得。常见的例子是 Rust 中的[元组](https://doc.rust-lang.org/std/primitive.tuple.html)以及[数组](https://doc.rust-lang.org/std/primitive.array.html)。在非自描述格式中，需要对`Vec<T>`的长度进行序列化，以便能够对`Vec<T>`进行反序列化。但是一个`[T; 16]`可以使用`serialize_tuple`进行序列化，因为长度在反序列化时已知，无需查看序列化的字节。

## 序列化结构体

Serde 区分四种类型的结构体。[普通结构体](https://doc.rust-lang.org/book/structs.html)和[元组结构体](https://doc.rust-lang.org/book/structs.html#tuple-structs)像序列或 map 一样遵循初始化，元素，结束的三个序列化步骤。[newtype 结构体](https://doc.rust-lang.org/book/structs.html#tuple-structs)和[单元结构体](https://doc.rust-lang.org/book/structs.html#unit-like-structs)更像原始类型。

```rust
// An ordinary struct. Use three-step process:
//   1. serialize_struct
//   2. serialize_field
//   3. end
struct Color {
    r: u8,
    g: u8,
    b: u8,
}

// A tuple struct. Use three-step process:
//   1. serialize_tuple_struct
//   2. serialize_field
//   3. end
struct Point2D(f64, f64);

// A newtype struct. Use serialize_newtype_struct.
struct Inches(u64);

// A unit struct. Use serialize_unit_struct.
struct Instance;
```

结构体和 map 在某些形式上可能看起来很相似，包括 JSON。区别是 Serde 在编译期间有一个常量字符串作为 key以便在反序列化时不需要再去查找序列化的数据。这种条件使某些数据格式可以比映射更有效，更紧凑的处理结构。

数据结构将 newtype 结构视为其内部值的不重要的封装器，仅对其内部之进行序列化。请查看示例：[JSON 处理 newtype 结构](https://serde.rs/json.html)

```rust
use serde::ser::{Serialize, Serializer, SerializeStruct};

struct Color {
    r: u8,
    g: u8,
    b: u8,
}

impl Serialize for Color {
    fn serialize<S>(&self, serializer: S) -> Result<S::Ok, S::Error>
    where
        S: Serializer,
    {
        // 3 is the number of fields in the struct.
        let mut state = serializer.serialize_struct("Color", 3)?;
        state.serialize_field("r", &self.r)?;
        state.serialize_field("g", &self.g)?;
        state.serialize_field("b", &self.b)?;
        state.end()
    }
}
```

## 序列化枚举

序列化枚举变体与序列化结构体很相似。

```rust
enum E {
    // Use three-step process:
    //   1. serialize_struct_variant
    //   2. serialize_field
    //   3. end
    Color { r: u8, g: u8, b: u8 },

    // Use three-step process:
    //   1. serialize_tuple_variant
    //   2. serialize_field
    //   3. end
    Point2D(f64, f64),

    // Use serialize_newtype_variant.
    Inches(u64),

    // Use serialize_unit_variant.
    Instance,
}
```

## 其他特殊的场景

`Serializer`特征还包含另外两种特殊情况。

`serialize_bytes`用来序列化`&[u8]`。一些格式将字节当作另一种序列，但是一些结构可以将字节序列化的更紧凑。目前在`Serialize`的实现中并没有将`serialize_bytes`用于`&[u8]`或者`Vec<u8>`的序列化，但是一旦[该规范](https://github.com/rust-lang/rust/issues/31844)应用于稳定的 Rust 时就会开始使用它。目前 [serde_bytes](https://docs.serde.rs/serde_bytes/) 库中的`serialize_bytes`可用于有效的处理`&[u8]`和`Vec<u8>`。

最后，`serialize_some`和`serialize_none`对应于`Option::Some`和`Option::None`。与其他枚举相比，用户对`Option`枚举的期望往往不同。Serde JSON 将会将`Option::None`序列化为`null`，将`Option::Some`序列化为其具体包含的值。