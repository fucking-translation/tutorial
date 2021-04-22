% 标记树聚束

```rust
macro_rules! call_a_or_b_on_tail {
    ((a: $a:expr, b: $b:expr), 调a $($tail:tt)*) => {
        $a(stringify!($($tail)*))
    };

    ((a: $a:expr, b: $b:expr), 调b $($tail:tt)*) => {
        $b(stringify!($($tail)*))
    };

    ($ab:tt, $_skip:tt $($tail:tt)*) => {
        call_a_or_b_on_tail!($ab, $($tail)*)
    };
}

fn compute_len(s: &str) -> Option<usize> {
    Some(s.len())
}

fn show_tail(s: &str) -> Option<usize> {
    println!("tail: {:?}", s);
    None
}

fn main() {
    assert_eq!(
        call_a_or_b_on_tail!(
            (a: compute_len, b: show_tail),
            规则的 递归部分 将 跳过 所有这些 标记
            它们 并不 关心 我们究竟 调b 还是 调a
            只有 终结规则 关心
        ),
        None
    );
    assert_eq!(
        call_a_or_b_on_tail!(
        (a: compute_len, b: show_tail),
        而现在 为了 显式 可能的路径 有两条
        我们也 调a 一哈: 它的 输入 应该
        自我引用 因此 我们给它 一个 72),
        Some(72)
    );
}
```

在十分复杂的递归宏中，可能需要非常多的参数，才足以在每层调用之间传递必要的标识符与表达式。然而，根据实现上的差异，可能存在许多这样的中间层，它们转发了这些参数，但并没有用到。

因此，将所有这些参数聚成一束，通过分组将其放进单独一棵标记树里；可以省事许多。这样一来，那些用不到这些参数的递归层可以直接捕获并替换这棵标记树，而不需要把整组参数完完全全准准确确地捕获替换掉。

上面的例子把表达式`$a`和`$b`聚束，然后作为一棵`tt`交由递归规则转发。随后，终结规则将这组标记打开，并访问其中的表达式。
