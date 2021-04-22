% 下推累积

```rust
macro_rules! init_array {
    (@accum (0, $_e:expr) -> ($($body:tt)*))
        => {init_array!(@as_expr [$($body)*])};
    (@accum (1, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (0, $e) -> ($($body)* $e,))};
    (@accum (2, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (1, $e) -> ($($body)* $e,))};
    (@accum (3, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (2, $e) -> ($($body)* $e,))};
    (@as_expr $e:expr) => {$e};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            init_array!(@accum ($n, e.clone()) -> ())
        }
    };
}

let strings: [String; 3] = init_array![String::from("hi!"); 3];
# assert_eq!(format!("{:?}", strings), "[\"hi!\", \"hi!\", \"hi!\"]");
```

在Rust中，所有宏最终必须展开为一个完整、有效的句法元素(比如表达式、条目等等)。这意味着，不可能定义一个最终展开为残缺构造的宏。

有些人可能希望，上例中的宏能被更加直截了当地表述成：

```ignore
macro_rules! init_array {
    (@accum 0, $_e:expr) => {/* empty */};
    (@accum 1, $e:expr) => {$e};
    (@accum 2, $e:expr) => {$e, init_array!(@accum 1, $e)};
    (@accum 3, $e:expr) => {$e, init_array!(@accum 2, $e)};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            [init_array!(@accum $n, e)]
        }
    };
}
```

他们预期的展开过程如下：

```ignore
            [init_array!(@accum 3, e)]
            [e, init_array!(@accum 2, e)]
            [e, e, init_array!(@accum 1, e)]
            [e, e, e]
```

然而，这一思路中，每个中间步骤的展开结果都是一个不完整的表达式。即便这些中间结果对外部来说绝不可见，Rust仍然禁止这种用法。

下推累积则使我们得以在完全完成之前毋需考虑构造的完整性，进而累积构建出我们所需的标记序列。顶端给出的示例中，宏调用的展开过程如下：

```ignore
init_array! { String:: from ( "hi!" ) ; 3 }
init_array! { @ accum ( 3 , e . clone (  ) ) -> (  ) }
init_array! { @ accum ( 2 , e.clone() ) -> ( e.clone() , ) }
init_array! { @ accum ( 1 , e.clone() ) -> ( e.clone() , e.clone() , ) }
init_array! { @ accum ( 0 , e.clone() ) -> ( e.clone() , e.clone() , e.clone() , ) }
init_array! { @ as_expr [ e.clone() , e.clone() , e.clone() , ] }
```

可以看到，每一步都在累积输出，直到规则完成，给出完整的表达式。

上述过程的关键点在于，使用`$($body:tt)*`来保存输出中间值，而不触发其它解析机制。采用`($input) -> ($output)`的形式仅是出于传统，用以明示此类宏的作用。

由于可以存储任意复杂的中间结果，下推累积在构建[标记树撕咬机](#incremental-tt-munchers)的过程中经常被用到。
