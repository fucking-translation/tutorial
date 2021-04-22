% 再探捕获与展开

一旦语法分析器开始消耗标记以匹配某捕获，整个过程便无法停止或回溯。这意味着，下述宏的第二项规则将永远无法被匹配到，无论输入是什么样的：

```rust
macro_rules! dead_rule {
    ($e:expr) => { ... };
    ($i:ident +) => { ... };
}
```

考虑当以`dead_rule!(x+)`形式调用此宏时，将会发生什么。解析器将从第一条规则开始试图进行匹配：它试图将输入解析为一个表达式；第一个标记(`x`)作为表达式是有效的，第二个标记——作为二元加的节点——在表达式中也是有效的。

至此，由于输入中并不包含二元加的右手侧元素，你可能会以为，分析器将会放弃尝试这一规则，转而尝试下一条规则。实则不然：分析器将会`panic`并终止整个编译过程，返回一个语法错误。

由于分析器的这一特点，下面这点尤为重要：一般而言，在书写宏规则时，应从最具体的开始写起，依次写至最不具体的。

为防范未来宏输入的解读方式改变所可能带来的句法影响，`macro_rules!`对各式捕获之后所允许的内容施加了诸多限制。在Rust1.3下，完整的列表如下：

* `item`: 任何标记
* `block`: 任何标记
* `stmt`: `=>` `、` `;`
* `pat`: `=>` `、` `=`、 `if`、 `in`
* `expr`: `=>` `、` `;`
* `ty`: `,`、 `=>`、 `:`、 `=`、 `>`、 `;`、 `as`
* `ident`: 任何标记
* `path`: `,`、 `=>`、 `:`、 `=`、 `>`、 `;`、 `as`
* `meta`: 任何标记
* `tt`: 任何标记

此外，`macro_rules!` 通常不允许一个重复紧跟在另一重复之后，即便它们的内容并不冲突。

有一条关于替换的特征经常引人惊奇：尽管看起来很像，但替换**并非**基于标记(token-based)的。下例展示了这一点：

```rust
macro_rules! capture_expr_then_stringify {
    ($e:expr) => {
        stringify!($e)
    };
}

fn main() {
    println!("{:?}", stringify!(dummy(2 * (1 + (3)))));
    println!("{:?}", capture_expr_then_stringify!(dummy(2 * (1 + (3)))));
}
```

注意到`stringify!`，这是一条内置的语法扩充，将所有输入标记结合在一起，作为单个字符串输出。

上述代码的输出将是：

```text
"dummy ( 2 * ( 1 + ( 3 ) ) )"
"dummy(2 * (1 + (3)))"
```

尽管二者的输入完全一致，它们的输出并不相同。这是因为，前者字符串化的是一列标记树，而后者字符串化的则是一个AST表达式节点。

我们另用一种方式展现二者的不同。第一种情况下，`stringify!`被调用时，输入是：

```text
«dummy» «(   )»
   ╭───────┴───────╮
    «2» «*» «(   )»
       ╭───────┴───────╮
        «1» «+» «(   )»
                 ╭─┴─╮
                  «3»
```

…而第二种情况下，`stringify!`被调用时，输入是：

```text
« »
 │ ┌─────────────┐
 └╴│ Call        │
   │ fn: dummy   │   ┌─────────┐
   │ args: ◌     │╶─╴│ BinOp   │
   └─────────────┘   │ op: Mul │
                   ┌╴│ lhs: ◌  │
        ┌────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
        │ LitInt │╶┘ └─────────┘ └╴│ BinOp   │
        │ val: 2 │                 │ op: Add │
        └────────┘               ┌╴│ lhs: ◌  │
                      ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────┐
                      │ LitInt │╶┘ └─────────┘ └╴│ LitInt │
                      │ val: 1 │                 │ val: 3 │
                      └────────┘                 └────────┘
```

如图所示，第二种情况下，输入仅有一棵标记树，它包含了一棵AST，这棵AST则是在解析`capture_expr_then_stringify!`的调用时，调用的输入经解析后所得的输出。因此，在这种情况下，我们最终看到的是字符串化AST节点所得的输出，而非字符串化标记所得的输出。

这一特征还有更加深刻的影响。考虑如下代码段：

```rust
macro_rules! capture_then_match_tokens {
    ($e:expr) => {match_tokens!($e)};
}

macro_rules! match_tokens {
    ($a:tt + $b:tt) => {"got an addition"};
    (($i:ident)) => {"got an identifier"};
    ($($other:tt)*) => {"got something else"};
}

fn main() {
    println!("{}\n{}\n{}\n",
        match_tokens!((caravan)),
        match_tokens!(3 + 6),
        match_tokens!(5));
    println!("{}\n{}\n{}",
        capture_then_match_tokens!((caravan)),
        capture_then_match_tokens!(3 + 6),
        capture_then_match_tokens!(5));
}
```

其输出将是：

```text
got an identifier
got an addition
got something else

got something else
got something else
got something else
```

因为输入被解析为AST节点，替换所得的结果将无法析构。也就是说，你没办法检查其内容，或是再按原先相符的匹配匹配它。

接下来这个例子尤其可能让人感到不解：

```rust
macro_rules! capture_then_what_is {
    (#[$m:meta]) => {what_is!(#[$m])};
}

macro_rules! what_is {
    (#[no_mangle]) => {"no_mangle attribute"};
    (#[inline]) => {"inline attribute"};
    ($($tts:tt)*) => {concat!("something else (", stringify!($($tts)*), ")")};
}

fn main() {
    println!(
        "{}\n{}\n{}\n{}",
        what_is!(#[no_mangle]),
        what_is!(#[inline]),
        capture_then_what_is!(#[no_mangle]),
        capture_then_what_is!(#[inline]),
    );
}
```

输出将是：

```text
no_mangle attribute
inline attribute
something else (# [ no_mangle ])
something else (# [ inline ])
```

得以幸免的捕获只有`tt`或`ident`两种。其余的任何捕获，一经替换，结果将只能被用于直接输出。
