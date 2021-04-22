% AST中的宏

如前所述，在Rust中，宏处理发生在AST生成**之后**。因此，调用宏的语法必须是Rust语言语法中规整相符的一部分。实际上，Rust语法包含数种“语法扩展”的形式。我们将它们同用例列出如下：

* `# [ $arg ]`; 如 `#[derive(Clone)]`, `#[no_mangle]`, …
* `# ! [ $arg ]`; 如 `#![allow(dead_code)]`, `#![crate_name="blang"]`, …
* `$name ! $arg`; 如 `println!("Hi!")`, `concat!("a", "b")`, …
* `$name ! $arg0 $arg1`; 如 `macro_rules! dummy { () => {}; }`.

头两种形式被称作“属性(attribute)”，被同时用于语言特属的结构(比如用于要求兼容C的ABI的`#[repr(C)]`)以及语法扩展(比如 `#[derive(Clone)]`)。当前没有办法定义这种形式的宏。

我们感兴趣的是第三种：我们通常使用的宏正是这种形式。注意，采用这种形式的并非只有宏：它是一种一般性的语法扩展形式。举例来说，`format!` 是宏，而`format_args!` (它被用于`format!`) 并不是。

第四种形式实际上宏无法使用。事实上，这种形式的唯一用例只有 `macro_rules!` 我们将在稍后谈到它。

将注意力集中到第三种形式 (`$name ! $arg`)上，我们的问题变成，对于每种可能的语法扩展，Rust的语法分析器(parser)如何知道`$arg`究竟长什么样?答案是它不需要知道。其实，提供给每次语法扩展调用的参数，是一棵标记树。具体来说，一棵非叶节点的标记树；即`(...)`，`[...]`，或`{...}`。拥有这一知识后，语法分析器如何理解如下调用形式，就变得显而易见了：

```rust
bitflags! {
    flags Color: u8 {
        const RED    = 0b0001,
        const GREEN  = 0b0010,
        const BLUE   = 0b0100,
        const BRIGHT = 0b1000,
    }
}

lazy_static! {
    static ref FIB_100: u32 = {
        fn fib(a: u32) -> u32 {
            match a {
                0 => 0,
                1 => 1,
                a => fib(a-1) + fib(a-2)
            }
        }

        fib(100)
    };
}

fn main() {
    let colors = vec![RED, GREEN, BLUE];
    println!("Hello, World!");
}
```

虽然看起来上述调用包含了各式各样的Rust代码，但对语法分析器来说，它们仅仅是堆毫无意义的标记树。为了让事情变得更清晰，我们把所有这些句法“黑盒”用⬚代替，仅剩下：

```ignore
bitflags! ⬚

lazy_static! ⬚

fn main() {
    let colors = vec! ⬚;
    println! ⬚;
}
```

再次重申，语法分析器对⬚不作任何假设；它记录黑盒所包含的标记，但并不尝试理解它们。

需要记下的点：

* Rust包含多种语法扩展。我们将仅仅讨论定义在`macro_rules!` 结构中的宏。
* 当遇见形如`$name! $arg`的结构时，该结构并不一定是宏，可能是其它语言扩展。
* 所有宏的输入都是非叶节点的单个标记树。
* 宏(其实所有一般意义上的语法扩展)都将作为抽象语法树的一部分被解析。

> **脚注**: 接下来(包括下一节)将提到的某些内容将适用于一般性的语法扩展。[^作者很懒]

[^作者很懒]: 这样比较方便，因为“宏”打起来比“语法扩展”更快更简单。

最后一点最为重要，它带来了一些深远的影响。由于宏将被解析进AST中，它们将仅仅只能出现在那些支持它们出现的位置。具体来说，宏能在如下位置出现：

* 模式(pattern)中
* 语句(statement)中
* 表达式(expression)中
* 条目(item)中
* `impl` 块中

一些并不支持的位置包括：

* 标识符(identifier)中
* `match`臂中
* 结构体的字段中
* 类型中[^类型宏]

[^类型宏]: 在非稳定Rust中可以通过`#![feature(type_macros)]`使用类型宏。见[Issue #27336](https://github.com/rust-lang/rust/issues/27336)。

绝对没有任何在上述位置以外的地方使用宏的可能。
