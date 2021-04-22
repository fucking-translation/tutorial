% 不是标识符的标识符

有两个标记，当你撞见时，很有可能最终认为它们是标识符，但实际上它们不是。然而正是这些标记，在某些情况下又的确是标识符。

第一个是`self`。毫无疑问，它是一个关键词。在一般的Rust代码中，不可能出现把它解读成标识符的情况；但在宏中这种情况则有可能发生：

```rust
macro_rules! what_is {
    (self) => {"the keyword `self`"};
    ($i:ident) => {concat!("the identifier `", stringify!($i), "`")};
}

macro_rules! call_with_ident {
    ($c:ident($i:ident)) => {$c!($i)};
}

fn main() {
    println!("{}", what_is!(self));
    println!("{}", call_with_ident!(what_is(self)));
}
```

上述代码的输出将是：

```text
the keyword `self`
the keyword `self`
```

但这没有任何道理！`call_with_ident!`要求一个标识符，而且它的确匹配到了，还成功替换了！所以，`self`同时是一个关键词，但又不是。你可能会想，好吧，但这鬼东西哪里重要呢？看看这个：

```rust
macro_rules! make_mutable {
    ($i:ident) => {let mut $i = $i;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_mutable!(self);
        self.0 *= 2;
        self
    }
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

编译它会失败，并报错：

```text
<anon>:2:28: 2:30 error: expected identifier, found keyword `self`
<anon>:2     ($i:ident) => {let mut $i = $i;};
                                    ^~
```

所以说，宏在匹配的时候，会欣然把`self`当作标识符接受，进而允许你把`self`带到那些实际上没办法使用的情况中去。但是，也成吧，既然得同时记住`self`既是关键词又是标识符，那下面这个**讲道理**应该可行，对吧？

```rust
macro_rules! make_self_mutable {
    ($i:ident) => {let mut $i = self;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_self_mutable!(mut_self);
        mut_self.0 *= 2;
        mut_self
    }
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

实际上也不行，编译错误变成：

```text
<anon>:2:33: 2:37 error: `self` is not available in a static method. Maybe a `self` argument is missing? [E0424]
<anon>:2     ($i:ident) => {let mut $i = self;};
                                         ^~~~
```

这同样也没有任何道理。它明明不在静态方法里。这简直就像是在抱怨说，它看见的两个`self`不是同一个`self`... 就搞得像关键词`self`也有卫生性一样，类似...标识符。

```rust
macro_rules! double_method {
    ($body:expr) => {
        fn double(mut self) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {{
        self.0 *= 2;
        self
    }}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

还是报同样的错。那这个如何：

```rust
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double(mut $self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {self, {
        self.0 *= 2;
        self
    }}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

终于管用了。所以说，`self`是关键词，但当它想的时候，它**同时**也能是一个标识符。那么，相同的道理对类似的其它东西有用吗？

```rust
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double($self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {_, 0}
}
# 
# fn main() {
#     println!("{:?}", Dummy(4).double().0);
# }
```

```text
<anon>:12:21: 12:22 error: expected ident, found _
<anon>:12     double_method! {_, 0}
                              ^
```

哈，当然不行。 `_`是一个关键词，在模式以及表达式中有效，但不知为何，并不像`self`，它并不是一个标识符；即便它——如同`self`——从定义上讲符合标识符的特性。

你可能觉得，既然`_`在模式中有效，那换成`$self_:pat`是不是就能一石二鸟了呢？可惜了，也不行，因为`self`不是一个有效的模式。真棒。

如果你真想同时匹配这两个标记，仅有的办法是换用`tt`来匹配。