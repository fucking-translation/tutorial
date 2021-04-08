## 使用 derive

Serde 提供了派生宏，用于为你的 crate 中定义的数据结构生成`Serialize`和`Derialize`实现，从而可以方便的以所有的 Serde 数据格式来表示它们。

**仅当你的代码中使用了`#[derive(Serialize, Deserialize)]`时才需要进行设置**。

这个功能基于 Rust 的`#[derive]`机制 (mechanism)，就像你用来自动派生内置的`Clone`，`Copy`，`Debug`或其他特征的实现一样。它能够为大多数(甚至包含复杂(elaborate) 范型或特征)的结构和枚举生成实现。在极少数情况下，你需要为特别复杂的类型[手动实现这两个特征](https://serde.rs/custom-serialization.html)。

这些派生需要 1.31 或更新的 Rust 编译器版本。

- [ ] 在 Cargo.toml 中添加`serde = { version = "1.0", features = ["derive"] }`依赖。
- [ ] 确保其他所有基于 Serde 的依赖(如`serde_json`) 与 Serde 1.0 兼容。
- [ ] 在你想要序列化的结构或枚举中，通过`use serde::Serialize`导入派生宏，对于相同模块中的结构体或枚举，你可以使用`#[derive(Serialize)]`。
- [ ] 同样的，使用`use serde::Deserialize`导入派生宏，并在你想要反序列化的结构体或枚举上添加`#[derive(Derialize)]`。

以下是`Cargo.toml`：

```toml
[package]
name = "my-crate"
version = "0.1.0"
authors = ["Me <user@rust-lang.org>"]

[dependencies]
serde = { version = "1.0", features = ["derive"] }

# serde_json is just for the example, not required in general
serde_json = "1.0"
```

现在，可以在`src/main.rs`中使用 Serde 自定义的派生实现：

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let point = Point { x: 1, y: 2 };

    let serialized = serde_json::to_string(&point).unwrap();
    println!("serialized = {}", serialized);

    let deserialized: Point = serde_json::from_str(&serialized).unwrap();
    println!("deserialized = {:?}", deserialized);
}
```

以下是输出：

```console
$ cargo run
serialized = {"x":1,"y":2}
deserialized = Point { x: 1, y: 2 }
```

## 故障排除

有时候你可能会看到编译错误，它告诉你：

```console
the trait `serde::ser::Serialize` is not implemented for `...`
```

即使结构体或枚举上已经添加了`#[derive(Serialize)]`。

这可能是因为你使用了与 Serde 版本不兼容的类库。你可能在 Cargo.toml 中依赖 serde 1.0，但是其他类库依赖于 serde 0.9。因此可能实现的是 serde 1.0 的`Serialize`特征，但是类库期望的是 serde 0.9 的`Serialize`实现。对于 Rust 编译器来说这是两个完全不同的特征。

解决办法是根据需要升级或降级类库直到 serde 的版本匹配为止。`cargo tree -d`命令有助于查找所有引入重复依赖项的位置。