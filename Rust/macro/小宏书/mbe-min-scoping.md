% 作用域

宏作用域的决定方式可能有一点反直觉。首先就与语言剩下的所有部分都不同的是，宏在子模组中仍然可见。

```rust
macro_rules! X { () => {}; }
mod a {
    X!(); // 已被定义
}
mod b {
    X!(); // 已被定义
}
mod c {
    X!(); // 已被定义
}
# fn main() {}
```

> **注意**：即使子模组的内容处在不同文件中，这些例子中所述的行为仍然保持不变。

其次，同样与语言剩下的所有部分不同，宏只有在其定义**之后**可见。下例展示了这一点。同时注意到，它也展示了宏不会“漏出”其定义所在的域：

```rust
mod a {
    // X!(); // 未被定义
}
mod b {
    // X!(); // 未被定义
    macro_rules! X { () => {}; }
    X!(); // 已被定义
}
mod c {
    // X!(); // 未被定义
}
# fn main() {}
```

需要阐明的是，即便宏定义被移至外围域，此顺序依赖行为仍旧不变：

```rust
mod a {
    // X!(); // 未被定义
}
macro_rules! X { () => {}; }
mod b {
    X!(); // 已被定义
}
mod c {
    X!(); // 已被定义
}
# fn main() {}
```

然而，对于宏们自身来说，此依赖行为不存在：

```rust
mod a {
    // X!(); // 未被定义
}
macro_rules! X { () => { Y!(); }; }
mod b {
    // X!(); // 已被定义, 但Y!未被定义
}
macro_rules! Y { () => {}; }
mod c {
    X!(); // 均已被定义
}
# fn main() {}
```

可通过 `#[macro_use]`属性将宏导出模组：

```rust
mod a {
    // X!(); // 未被定义
}
#[macro_use]
mod b {
    macro_rules! X { () => {}; }
    X!(); // 已被定义
}
mod c {
    X!(); // 已被定义
}
# fn main() {}
```

注意到这一特性可能会产生一些奇怪的后果，因为宏中的标识符只有在宏展开的过程中才会被解析。

```rust
mod a {
    // X!(); // 未被定义
}
#[macro_use]
mod b {
    macro_rules! X { () => { Y!(); }; }
    // X!(); // 已被定义，但Y!并未被定义
}
macro_rules! Y { () => {}; }
mod c {
    X!(); // 均已被定义
}
# fn main() {}
```

让情形变得更加复杂的是，当`#[macro_use]`被作用于`extern crate`时，其行为又会发生进一步变化：此类声明从效果上看，类似于被放在了整个模组的顶部。因此，假设在某个`extern crate mac`中定义了`X!`，则有：

```rust
mod a {
    // X!(); // 已被定义，但Y!并未被定义
}
macro_rules! Y { () => {}; }
mod b {
    X!(); // 均已被定义
}
#[macro_use] extern crate macs;
mod c {
    X!(); // 均已被定义
}
# fn main() {}
```

最后，注意这些有关作用域的行为同样适用于函数，除了`#[macro_use]`以外(它并不适用):

```rust
macro_rules! X {
    () => { Y!() };
}

fn a() {
    macro_rules! Y { () => {"Hi!"} }
    assert_eq!(X!(), "Hi!");
    {
        assert_eq!(X!(), "Hi!");
        macro_rules! Y { () => {"Bye!"} }
        assert_eq!(X!(), "Bye!");
    }
    assert_eq!(X!(), "Hi!");
}

fn b() {
    macro_rules! Y { () => {"One more"} }
    assert_eq!(X!(), "One more");
}
# 
# fn main() {
#     a();
#     b();
# }
```

由于前述种种规则，一般来说，建议将所有应对整个`crate`均可见的宏的定义置于根模组的最顶部，借以确保它们一直可用。
