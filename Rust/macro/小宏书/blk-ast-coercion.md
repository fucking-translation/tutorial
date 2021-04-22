% AST强转

在替换`tt`时，Rust的解析器并不十分可靠。当它期望得到某类特定的语法构造时，如果摆在它面前的是一坨替换后的`tt`标记，就有可能出现问题。解析器常常直接选择死亡，而非尝试去解析它们。在这类情况中，就要用到AST强转。

```rust
# #![allow(dead_code)]
# 
macro_rules! as_expr { ($e:expr) => {$e} }
macro_rules! as_item { ($i:item) => {$i} }
macro_rules! as_pat  { ($p:pat) =>  {$p} }
macro_rules! as_stmt { ($s:stmt) => {$s} }
# 
# as_item!{struct Dummy;}
# 
# fn main() {
#     as_stmt!(let as_pat!(_) = as_expr!(42));
# }
```

这些强制变换经常与[下推累积](pat-push-down-accumulation.md)宏一同使用，以使解析器能够将最终输出的`tt`序列当作某类特定的语法构造对待。

注意，之所以只有这几种强转宏，是因为决定可用强转类型的点在于宏可展开在哪些地方，而不是宏能够捕捉哪些东西。也就是说，因为宏没办法在·`type`处展开[issue-27245](https://github.com/rust-lang/rust/issues/27245)，所以就没办法实现什么`as_ty!`强转。
