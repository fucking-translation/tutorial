% 导入/导出

有两种将宏暴露给更广范围的方法。第一种是采用`#[macro_use]`属性。它不仅适用于模组，同样适用于`extern crate`。例如：

```rust
#[macro_use]
mod macros {
    macro_rules! X { () => { Y!(); } }
    macro_rules! Y { () => {} }
}

X!();
#
# fn main() {}
```

可通过`#[macro_export]`将宏从当前`crate`导出。注意，这种方式无视所有可见性设定。

定义库包`macs`如下：

```rust
mod macros {
    #[macro_export] macro_rules! X { () => { Y!(); } }
    #[macro_export] macro_rules! Y { () => {} }
}

// X!和Y!并非在此处定义的，但它们**的确**被
// 导出了，即便macros并非pub。
```

则下述代码将成立：

```rust
X!(); // X!已被定义
#[macro_use] extern crate macs;
X!();
# 
# fn main() {}
```

注意只有在根模组中，才可将`#[macro_use]`用于`extern crate`。

最后，在从`extern crate`导入宏时，可显式控制导入**哪些**宏。可利用这一特性来限制命名空间污染，或是覆写某些特定的宏。就像这样：

```rust
// 只导入`X!`这一个宏
#[macro_use(X)] extern crate macs;

// X!(); // X!已被定义，但Y!未被定义

macro_rules! Y { () => {} }

X!(); // 均已被定义

fn main() {}
```

当导出宏时，常常出现的情况是，宏定义需要其引用所在`crate`内的非宏符号。由于`crate`可能被重命名等，我们可以使用一个特殊的替换变量：`$crate`。它总将被扩展为宏定义所在的`crate`在当前上下文中的绝对路径(比如 `:: macs`)。

注意这招并不适用于宏，因为通常名称的决定进程并不适用于宏。也就是说，你没办法采用类似`$crate::Y!`的代码来引用某个自己`crate`里的特定宏。结合采用`#[macro_use]`做到的选择性导入，我们得出：在宏被导入进其它`crate`时，当前没有办法保证其定义中的其它任一给定宏也一定可用。

推荐的做法是，在引用非宏名称时，总是采用绝对路径。这样可以最大程度上避免冲突，包括跟标准库中名称的冲突。
