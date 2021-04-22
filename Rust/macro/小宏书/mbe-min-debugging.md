% 调试

`rustc`提供了一些工具用来调试宏。其中，最有用的之一是`trace_macros!`。它会指示编译器，在每一个宏调用被展开之前将其转印出来。例如，给定下列代码：

```rust
# // Note: make sure to use a nightly channel compiler.
#![feature(trace_macros)]

macro_rules! each_tt {
    () => {};
    ($_tt:tt $($rest:tt)*) => {each_tt!($($rest)*);};
}

each_tt!(foo bar baz quux);
trace_macros!(true);
each_tt!(spim wak plee whum);
trace_macros!(false);
each_tt!(trom qlip winp xod);
# 
# fn main() {}
```

编译输出将包含：

```text
each_tt! { spim wak plee whum }
each_tt! { wak plee whum }
each_tt! { plee whum }
each_tt! { whum }
each_tt! {  }
```

它在调试递归很深的宏时尤其有用。同时，它可以在命令提示符中被打开，在编译指令中附加`-Z trace-macros`即可。

另一有用的宏是`log_syntax!`。它将使得编译器输出所有经过编译器处理的标记。举个例子，下述代码可以让编译器唱一首歌：

```rust
# // Note: make sure to use a nightly channel compiler.
#![feature(log_syntax)]

macro_rules! sing {
    () => {};
    ($tt:tt $($rest:tt)*) => {log_syntax!($tt); sing!($($rest)*);};
}

sing! {
    ^ < @ < . @ *
    '\x08' '{' '"' _ # ' '
    - @ '$' && / _ %
    ! ( '\t' @ | = >
    ; '\x08' '\'' + '$' ? '\x7f'
    , # '"' ~ | ) '\x07'
}
# 
# fn main() {}
```

比起`trace_macros!`来说，它能够做一些更有针对性的调试。

有时问题出在宏展开后的结果里。对于这种情况，可用编译命令`--pretty`来勘察。给出下列代码：

```rust
// Shorthand for initialising a `String`.
macro_rules! S {
    ($e:expr) => {String::from($e)};
}

fn main() {
    let world = S!("World");
    println!("Hello, {}!", world);
}
```

并用如下编译命令进行编译，

```shell
rustc -Z unstable-options --pretty expanded hello.rs
```

将输出如下内容(略经修改以符合排版)：

```rust
#![feature(no_std, prelude_import)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
// Shorthand for initialising a `String`.
fn main() {
    let world = String::from("World");
    ::std::io::_print(::std::fmt::Arguments::new_v1(
        {
            static __STATIC_FMTSTR: &'static [&'static str]
                = &["Hello, ", "!\n"];
            __STATIC_FMTSTR
        },
        &match (&world,) {
             (__arg0,) => [
                ::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)
            ],
        }
    ));
}
```

`--pretty`还有其它一些可用选项，可通过`rustc -Z unstable-options --help -v`来列出。此处并不提供该选项表；因为，正如指令本身所暗示的，表中的一切内容在任何时间点都有可能发生改变。
