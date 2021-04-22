% Ook!

此宏是对[Ook!密文](http://www.dangermouse.net/esoteric/ook.html)的实现，该语言与[Brainfuck密文](http://www.muppetlabs.com/~breadbox/bf/)同构。

此语言的执行模式非常简单：内存被表示为一列总量不定(通常至少包括30,000个)的“单元”(每个单元至少8bit)；另有一个指向该内存的指针，最初指向位置0；还有一个执行栈(用来实现循环)和一个指向程序代码的指针。最后两个组件并未暴露给程序本身，它们属于程序的运行时性质。

语言本身仅由三种标记，`Ook.`、`Ook?`及`Ook!`构成。它们两两组合，构成了八种运算符：

* `Ook. Ook?` - 指针增。
* `Ook? Ook.` - 指针减。
* `Ook. Ook.` - 所指单元增。
* `Ook! Ook!` - 所指单元减。
* `Ook! Ook.` - 将所指单元写至标准输出。
* `Ook. Ook!` - 从标准输入读至所指单元。
* `Ook! Ook?` - 进入循环。
* `Ook? Ook!` - 如果所指单元非零，跳回循环起始位置；否则，继续执行。

Ook!之所以有趣，是因为它图灵完备。这意味着，你必须要在同样图灵完备的环境中才能实现它。

## 实现

```ignore
#![recursion_limit = "158"]
```

实际上，这个值将是我们随后给出的示例程序编译成功所需的最低值。如果你很好奇究竟是怎样的程序，会如此这般复杂，以至于必须要把递归极限调至默认值的近五倍大... [请大胆猜测](https://en.wikipedia.org/wiki/Hello_world_program)。

```ignore
type CellType = u8;
const MEM_SIZE: usize = 30_000;
```

加入这些定义，以供宏展开使用[^*]

[^*]: 我们的确可以在宏内部定义这些，但那样一来它们就必须被显式的传来传去(由于宏的卫生性)。坦率地讲，当我意识到真的需要这些定义的时候，我的宏已经写得七七八八了...如果没有绝对的必要，你想跟我一样尝试修补那样的烂摊子吗？

```ignore
macro_rules! Ook {
```

可能应该取名`ook!`以符合Rust标准的命名传统。然而良机不可错失，我们就用本名吧！

我们使用了[内用规则](pat-internal-rules.md)模式；此宏的规则因而可被分为几个模块。

第一块是`@start`规则，负责为之后的展开搭建舞台。没什么特别的地方：先定义一些变量、效用函数，然后处理展开的大头。

一些脚注：

* 我们之所以选择展开为函数，很大原因在于，这样以来就可以采用`try!`来简化错误处理流程。
* 有些时候，举例来说，如果用户想写一个不做I/O的程序，那么I/O相关的名称就不会被用到。我们让部分名称以`_`起头，正是为了使编译器不对此类情况产生抱怨。

```ignore
    (@start $($Ooks:tt)*) => {
        {
            fn ook() -> ::std::io::Result<Vec<CellType>> {
                use ::std::io;
                use ::std::io::prelude::*;
    
                fn _re() -> io::Error {
                    io::Error::new(
                        io::ErrorKind::Other,
                        String::from("ran out of input"))
                }
                
                fn _inc(a: &mut [u8], i: usize) {
                    let c = &mut a[i];
                    *c = c.wrapping_add(1);
                }
                
                fn _dec(a: &mut [u8], i: usize) {
                    let c = &mut a[i];
                    *c = c.wrapping_sub(1);
                }
    
                let _r = &mut io::stdin();
                let _w = &mut io::stdout();
        
                let mut _a: Vec<CellType> = Vec::with_capacity(MEM_SIZE);
                _a.extend(::std::iter::repeat(0).take(MEM_SIZE));
                let mut _i = 0;
                {
                    let _a = &mut *_a;
                    Ook!(@e (_a, _i, _inc, _dec, _r, _w, _re); ($($Ooks)*));
                }
                Ok(_a)
            }
            ook()
        }
    };
```

### 解析运算符码

接下来，是一列“执行”规则，用于从输入中解析运算符码。

这列规则的通用形式是`(@e $syms; ($input))`。从`@start`规则种我们可以看出，`$syms`包含了实现Ook程序所必须的符号：输入、输出、内存等等。我们使用了[标记树聚束](pat-tt-bundling.md)来简化转发这些符号的流程。

第一条规则是终止规则：一旦没有更多输入，我们就停下来。

```ignore
    (@e $syms:tt; ()) => {};
```

其次，是一些适用于绝大部分运算符码的规则：我们剥下运算符码，换上其相应的Rust代码，然后继续递归处理输入剩下的部分。教科书式的[标记树撕咬机](pat-incremental-tt-munchers.md)。

```ignore
    // 指针增
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook? $($tail:tt)*))
    => {
        $i = ($i + 1) % MEM_SIZE;
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // 指针减
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook? Ook. $($tail:tt)*))
    => {
        $i = if $i == 0 { MEM_SIZE } else { $i } - 1;
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // 所指增
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook. $($tail:tt)*))
    => {
        $inc($a, $i);
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // 所指减
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook! $($tail:tt)*))
    => {
        $dec($a, $i);
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // 输出
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook. $($tail:tt)*))
    => {
        try!($w.write_all(&$a[$i .. $i+1]));
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
    
    // 读入
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook. Ook! $($tail:tt)*))
    => {
        try!(
            match $r.read(&mut $a[$i .. $i+1]) {
                Ok(0) => Err($re()),
                ok @ Ok(..) => ok,
                err @ Err(..) => err
            }
        );
        Ook!(@e ($a, $i, $inc, $dec, $r, $w, $re); ($($tail)*));
    };
```

现在要弄复杂的了。运算符码`Ook! Ook?`标记着循环的开始。Ook!中的循环，翻译成Rust代码的话，类似：

> **注意**：这不是宏定义的组成部分。
>
> ```ignore
> while memory[ptr] != 0 {
>     // 循环内容
> }
> ```

显然，我们无法为中间步骤生成不完整的循环。这一问题似可用[下推累积](pat-push-down-accumulation.md)解决，但更根本的困难在于，我们无论如何都没办法生成类似`while memory[ptr] != {`的中间结果，完全不可能。因为它引入了不匹配的花括号。

解决方法是，把输入分成两部分：循环内部的，循环之后的。前者由`@x`规则处理，后者由`@s`处理。

```ignore
    (@e ($a:expr, $i:expr, $inc:expr, $dec:expr, $r:expr, $w:expr, $re:expr);
        (Ook! Ook? $($tail:tt)*))
    => {
        while $a[$i] != 0 {
            Ook!(@x ($a, $i, $inc, $dec, $r, $w, $re); (); (); ($($tail)*));
        }
        Ook!(@s ($a, $i, $inc, $dec, $r, $w, $re); (); ($($tail)*));
    };
```

### 提取循环区块

接下来是“提取”规则组`@x`。它们负责接受输入尾，并将之展开为循环的内容。这组规则的一般形式为：`(@x $sym; $depth; $buf; $tail)`。

`$sym`的用处与上相同。`$tail`表示需要被解析的输入；而`$buf`则作为[下推累积的缓存](pat-push-down-accumulation.md)，循环内的运算符码在经过解析后将被存入其中。那么，`$depth`代表什么？

目前为止，我们还未提及如何处理嵌套循环。`$depth`的作用正在于此：我们需要记录当前循环在整个嵌套之中的深度，同时保证解析不会过早或过晚终止，而是刚好停在恰当的位置。[^justright]

[^justright]: 一个鲜有人知的事实[^fact]是，金发姑娘的故事实际上是一则写给准确词法解析技术的寓言。

[^fact]: 这个“事实”实际上意思是“臭不要脸的谣言”。

由于在宏中没办法进行计算，而将数目匹配规则一一列出又不太可行(想想下面这一整套规则都得复制粘贴一堆不算小的整数的话，会是什么样子)，我们将只好回头采用最古老最珍贵的计数方法之一：亲手去数。

当然了，宏没有手，我们实际采用的是[算盘计数模式](pat-provisional.md#算盘计数)。具体来说，我们选用标记`@`，每个`@`都表示新的一层嵌套。把这些`@`们放进一组后，我们就可以实现所需的操作了：

* 增加层数：匹配`($($depth:tt)*)`并用`(@ $($depth)*)`替换。
* 减少层数：匹配`(@ $($depth:tt)*)`并用`($($depth)*)`替换。
* 与0相比较：匹配`()`。

规则组中的第一条规则，用于在找到 `Ook? Ook!`输入序列时，终止当前循环体的解析。随后，我们需要把累积所得的循环体内容发给先前定义的`@e`组规则。

注意，规则对于输入所剩的尾部不作任何处理(这项工作将由`@s`组的规则完成)。

```ignore
    (@x $syms:tt; (); ($($buf:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        // 最外层的循环已被处理完毕，现在转而处理缓存到的标记。
        Ook!(@e $syms; ($($buf)*));
    };
```

紧接着，是负责进出嵌套的一些规则。它们修改深度计数，并将运算符码放入缓存。

```ignore
    (@x $syms:tt; ($($depth:tt)*); ($($buf:tt)*);
        (Ook! Ook? $($tail:tt)*))
    => {
        // 嵌套变深
        Ook!(@x $syms; (@ $($depth)*); ($($buf)* Ook! Ook?); ($($tail)*));
    };
    
    (@x $syms:tt; (@ $($depth:tt)*); ($($buf:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        // 嵌套变浅
        Ook!(@x $syms; ($($depth)*); ($($buf)* Ook? Ook!); ($($tail)*));
    };
```

最后剩下的所有情况将交由一条规则处理。注意到它用的`$op0`和`$op1`两处捕获；对于Rust来说，Ook!中的一个标记将被视作两个标记：标识符`Ook`与剩下的符号。因此，我们用此规则来处理其它任何Ook!的非循环运算符，将`!`, `?`和`.`作为`tt`匹配，并捕获之。

我们放置`$depth`，仅将运算符码推至缓存区中。

```ignore
    (@x $syms:tt; $depth:tt; ($($buf:tt)*);
        (Ook $op0:tt Ook $op1:tt $($tail:tt)*))
    => {
        Ook!(@x $syms; $depth; ($($buf)* Ook $op0 Ook $op1); ($($tail)*));
    };
```

### 跳过循环区块

这组规则与循环提取大致相同，不过它们并不关心循环的内容(也因此不需要累积缓存)。它们仅仅关心循环何时被完全跳过。彼时，我们将恢复到`@e`组规则中并继续处理剩下的输入。

因此，我们将不加进一步说明地列出它们：

```ignore
    // End of loop.
    (@s $syms:tt; ();
        (Ook? Ook! $($tail:tt)*))
    => {
        Ook!(@e $syms; ($($tail)*));
    };

    // Enter nested loop.
    (@s $syms:tt; ($($depth:tt)*);
        (Ook! Ook? $($tail:tt)*))
    => {
        Ook!(@s $syms; (@ $($depth)*); ($($tail)*));
    };
    
    // Exit nested loop.
    (@s $syms:tt; (@ $($depth:tt)*);
        (Ook? Ook! $($tail:tt)*))
    => {
        Ook!(@s $syms; ($($depth)*); ($($tail)*));
    };

    // Not a loop opcode.
    (@s $syms:tt; ($($depth:tt)*);
        (Ook $op0:tt Ook $op1:tt $($tail:tt)*))
    => {
        Ook!(@s $syms; ($($depth)*); ($($tail)*));
    };
```

### 入口

这是唯一一条非内用规则。

需注意的一点是，由于此规则单纯地匹配*所有*提供的标记，它极其危险。任何错误输入，都将造成其上的内用规则匹配完全失败，进而又落至匹配它(成功)的后果；引发无尽递归。

当在写、改及调试此类宏的过程中，明智的做法是，在此类规则的匹配头部加上临时性前缀，比如给此例加上一个`@entry`；以防止无尽递归，并得到更加恰当有效的错误信息。

```ignore
    ($($Ooks:tt)*) => {
        Ook!(@start $($Ooks)*)
    };
}
```

### 用例

现在终于是时候上测试了。

```ignore
fn main() {
    let _ = Ook!(
        Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook! Ook?  Ook? Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook?  Ook! Ook!  Ook? Ook!  Ook? Ook.
        Ook! Ook.  Ook. Ook?  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook?
        Ook! Ook!  Ook? Ook!  Ook? Ook.  Ook. Ook.
        Ook! Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook! Ook.  Ook! Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook.  Ook. Ook?  Ook. Ook?
        Ook. Ook?  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook?
        Ook! Ook!  Ook? Ook!  Ook? Ook.  Ook! Ook.
        Ook. Ook?  Ook. Ook?  Ook. Ook?  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook! Ook?  Ook? Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook?  Ook! Ook!  Ook? Ook!  Ook? Ook.
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.
        Ook? Ook.  Ook? Ook.  Ook? Ook.  Ook? Ook.
        Ook! Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook! Ook.  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook.
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook!  Ook! Ook!  Ook! Ook!  Ook! Ook!
        Ook! Ook.  Ook. Ook?  Ook. Ook?  Ook. Ook.
        Ook! Ook.  Ook! Ook?  Ook! Ook!  Ook? Ook!
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook. Ook.  Ook. Ook.
        Ook. Ook.  Ook. Ook.  Ook! Ook.
    );
}
```

运行(在编译器进行数百次递归宏展开而停顿相当长一段时间之后)的输出将是：

```text
Hello World!
```

由此，我们揭示出了令人惊恐的真相：`macro_rules!`是图灵完备的！

### 附注

此文所基的宏，是一个名为“Hodor!”的同构语言实现。Manish Goregaokar后来[用那个宏实现了一个Brainfuck的解析器](https://www.reddit.com/r/rust/comments/39wvrm/hodor_esolang_as_a_rust_macro/cs76rqk?context=10000)。也就是说，那个Brainfuck解析器用`Hodor!`宏写成，而后者本身则又是由`macro_rules!`实现的！

传说在把递归极限提至*三百万*，并让之编译了*四天*后，整个过程终于得以完成。

...收场的方式是栈溢出中止。时至今日，Rust宏驱动的密文编程语言仍然绝非可行的开发手段。
