% 标记树撕咬机

```rust
macro_rules! mixed_rules {
    () => {};
    (trace $name:ident; $($tail:tt)*) => {
        {
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
    (trace $name:ident = $init:expr; $($tail:tt)*) => {
        {
            let $name = $init;
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
}
# 
# fn main() {
#     let a = 42;
#     let b = "Ho-dee-oh-di-oh-di-oh!";
#     let c = (false, 2, 'c');
#     mixed_rules!(
#         trace a;
#         trace b;
#         trace c;
#         trace b = "They took her where they put the crazies.";
#         trace b;
#     );
# }
```

此模式可能是最强大的宏解析技巧。通过使用它，一些极其复杂的语法都能得到解析。

“标记树撕咬机”是一种递归宏，其工作机制有赖于对输入的顺次、逐步处理。处理过程的每一步中，它都将匹配并移除(“撕咬”掉)输入头部的一列标记，得到一些中间结果，然后再递归地处理输入剩下的尾部。

名称中含有“标记树”，是因为输入中尚未被处理的部分总是被捕获在`$($tail:tt)*`的形式中。之所以如此，是因为只有通过使用`tt`的重复才能做到无损地捕获住提供给宏的部分输入。

标记树撕咬机仅有的限制，也是整个宏系统的局限：

* 你只能匹配`macro_rules!`允许匹配的字面值和语法结构。
* 你无法匹配不成对的标记组(unbalanced group)。

然而，需要把宏递归的局限性纳入考量。`macro_rules!`没有做任何形式的尾递归消除或优化。在写标记树撕咬机时，推荐多花些功夫，尽可能地限制递归调用的次数。对于输入的变化，增加额外的匹配分支(而非采用中间层并使用递归)；或对输入句法施加限制，以便于对标准重复的记录追踪。
