% 回调

```rust
macro_rules! call_with_larch {
    ($callback:ident) => { $callback!(larch) };
}

macro_rules! expand_to_larch {
    () => { larch };
}

macro_rules! recognise_tree {
    (larch) => { println!("#1, 落叶松。") };
    (redwood) => { println!("#2, THE巨红杉。") };
    (fir) => { println!("#3, 冷杉。") };
    (chestnut) => { println!("#4, 七叶树。") };
    (pine) => { println!("#5, 欧洲赤松。") };
    ($($other:tt)*) => { println!("不懂，可能是种桦树？") };
}

fn main() {
    recognise_tree!(expand_to_larch!());
    call_with_larch!(recognise_tree);
}
```

由于宏展开的机制限制，(在Rust1.2中)不可能做到把一例宏的展开结果作为有效信息提供给另一例宏。这为宏的模组化工作施加了难度。

使用递归并传递回调是条出路。作为演示，上例两处宏调用的展开过程如下：

```ignore
recognise_tree! { expand_to_larch ! (  ) }
println! { "I don't know; some kind of birch maybe?" }
// ...

call_with_larch! { recognise_tree }
recognise_tree! { larch }
println! { "#1, the Larch." }
// ...
```

可以使用`tt`的重复来将任意参数转发给回调：

```rust
macro_rules! callback {
    ($callback:ident($($args:tt)*)) => {
        $callback!($($args)*)
    };
}

fn main() {
    callback!(callback(println("Yes, this *was* unnecessary.")));
}
```

如有需要，当然还可以在参数中增加额外的标记。
