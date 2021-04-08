## 属性

[属性](https://doc.rust-lang.org/book/attributes.html)用于自定义 Serde 派生生成的`Serialize`和`Deserialize`实现。这需要 1.15 甚至更高版本的 Rust 编译器。

这里有三种类型的属性：

- [容器属性](https://serde.rs/container-attrs.html) - 应用于结构体或枚举的声明。
- [变量属性](https://serde.rs/variant-attrs.html) - 应用于枚举的变量。
- [字段属性](https://serde.rs/field-attrs.html) - 应用于结构体的一个字段或者一个枚举变量。

```rust
#[derive(Serialize, Deserialize)]
#[serde(deny_unknown_fields)]  // <-- this is a container attribute
struct S {
    #[serde(default)]  // <-- this is a field attribute
    f: i32,
}

#[derive(Serialize, Deserialize)]
#[serde(rename = "e")]  // <-- this is also a container attribute
enum E {
    #[serde(rename = "a")]  // <-- this is a variant attribute
    A(String),
}
```

请注意，单个结构体，枚举，变量或者字段可能具有多个属性。