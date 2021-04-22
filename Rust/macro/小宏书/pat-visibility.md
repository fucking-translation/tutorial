% 可见性

在Rust中，因为没有类似`vis`的匹配选项，匹配替换可见性标记比较难搞。

## 匹配与忽略

根据上下文，可由重复做到这点：

```rust
macro_rules! struct_name {
    ($(pub)* struct $name:ident $($rest:tt)*) => { stringify!($name) };
}
# 
# fn main() {
#     assert_eq!(struct_name!(pub struct Jim;), "Jim");
# }
```

上例将匹配公共可见或本地可见的`struct`条目。但它还能匹配到`pub pub` (十分公开?)甚至是`pub pub pub pub` (真的非常非常公开)。防止这种情况出现的最好方法，只有祈祷调用方没那么多毛病。

## 匹配和替换

由于不能将重复的内容和其自身同时绑定至一个变量，没有办法将`$(pub)*`的内容直接拿去替换使用。因此，我们只好使用多条规则：

```rust
macro_rules! newtype_new {
    (struct $name:ident($t:ty);) => { newtype_new! { () struct $name($t); } };
    (pub struct $name:ident($t:ty);) => { newtype_new! { (pub) struct $name($t); } };
    
    (($($vis:tt)*) struct $name:ident($t:ty);) => {
        as_item! {
            impl $name {
                $($vis)* fn new(value: $t) -> Self {
                    $name(value)
                }
            }
        }
    };
}

macro_rules! as_item { ($i:item) => {$i} }
# 
# #[derive(Debug, Eq, PartialEq)]
# struct Dummy(i32);
# 
# newtype_new! { struct Dummy(i32); }
# 
# fn main() {
#     assert_eq!(Dummy::new(42), Dummy(42));
# }
```

> **参考**：[AST强转](blk-ast-coercion.md).

这里，我们用到了宏对成组的任意标记的匹配能力，来同时匹配`()`与`(pub)`，并将所得内容替换到输出中。因为在此处解析器不会期望看到一个`tt`重复的展开结果，我们需要使用[AST强转](blk-ast-coercion.md)来使代码正常运作。
