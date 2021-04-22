% 宏，实践介绍

本章节将通过一个相对简单、可行的例子来介绍Rust的“示例宏”系统。我们将不会试图解释整个宏系统错综复杂的构造；而是试图让读者能够舒适地了解宏的书写方式，以及为何如斯。

在[Rust官方教程中也有一章讲解宏](http://doc.rust-lang.org/book/macros.html)([中文版](https://kaisery.gitbooks.io/rust-book-chinese/content/content/Macros%20%E5%AE%8F.html))，同样提供了高层面的讲解。同时，本书也有一章[更富条理的介绍](mbe-README.html)，旨在详细阐释宏系统。

## 一点背景知识

> **注意**：别慌！我们通篇只会涉及到下面这一点点数学。如果想直接看重点，本小节可被安全跳过。

如果你不了解，所谓“递推(recurrence)关系”是指这样一个序列，其中的每个值都由先前的一个或多个值决定，并最终由一个或多个初始值完全决定。举例来说，[Fibonacci数列](https://en.wikipedia.org/wiki/Fibonacci_number)可被定义为如下关系：

<!-- MATH START: $F_n = 0, 1, \ldots, F_n-1 + F_n - 2$ -->
<style type="text/css">
    .katex {
        font: 400 1.21em/1.2 KaTeX_Main;
        white-space: nowrap;
        font-family: "Cambria Math", "Cambria", serif;
    }
    
    .katex .vlist > span > {
        display: inline-block;
    }
    
    .mathit {
        font-style: italic;
    }
    
    .katex .reset-textstyle.scriptstyle {
        font-size: 0.7em;
    }
    
    .katex .reset-textstyle.textstyle {
        font-size: 1em;
    }
    
    .katex .textstyle > .mord + .mrel {
        margin-left: 0.27778em;
    }
    
    .katex .textstyle > .mrel + .minner, .katex .textstyle > .mrel + .mop, .katex .textstyle > .mrel + .mopen, .katex .textstyle > .mrel + .mord {
        margin-left: 0.27778em;
    }
    
    .katex .textstyle > .mclose + .minner, .katex .textstyle > .minner + .mop, .katex .textstyle > .minner + .mord, .katex .textstyle > .mpunct + .mclose, .katex .textstyle > .mpunct + .minner, .katex .textstyle > .mpunct + .mop, .katex .textstyle > .mpunct + .mopen, .katex .textstyle > .mpunct + .mord, .katex .textstyle > .mpunct + .mpunct, .katex .textstyle > .mpunct + .mrel {
        margin-left: 0.16667em;
    }
    
    .katex .textstyle > .mord + .mbin {
        margin-left: 0.22222em;
    }
    
    .katex .textstyle > .mbin + .minner, .katex .textstyle > .mbin + .mop, .katex .textstyle > .mbin + .mopen, .katex .textstyle > .mbin + .mord {
        margin-left: 0.22222em;
    }
</style>

<div class="katex" style="font-size: 100%; text-align: center;">
    <span class="katex"><span class="katex-inner"><span style="height: 0.68333em;" class="strut"></span><span style="height: 0.891661em; vertical-align: -0.208331em;" class="strut bottom"></span><span class="base textstyle uncramped"><span class="reset-textstyle displaystyle textstyle uncramped"><span class="mord displaystyle textstyle uncramped"><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord mathit">n</span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mrel">=</span><span class="mord">0</span><span class="mpunct">,</span><span class="mord">1</span><span class="mpunct">,</span><span class="mpunct">…</span><span class="mpunct">,</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">−</span><span class="mord">1</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span><span class="mbin">+</span><span class="mord"><span class="mord mathit" style="margin-right: 0.13889em;">F</span><span class="vlist"><span style="top: 0.15em; margin-right: 0.05em; margin-left: -0.13889em;" class=""><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span><span class="reset-textstyle scriptstyle cramped"><span class="mord scriptstyle cramped"><span class="mord mathit">n</span><span class="mbin">-</span><span class="mord">2</span></span></span></span><span class="baseline-fix"><span class="fontsize-ensurer reset-size5 size5"><span style="font-size: 0em;" class="">​</span></span>​</span></span></span></span></span></span></span></span>
</div>
<!-- MATH END -->

即，序列的前两个数分别为0和1，而第3个则为<em>F<sub>0</sub></em> + <em>F<sub>1</sub></em> = 0 + 1 = 1，第4个为<em>F<sub>1</sub></em> + <em>F<sub>2</sub></em> = 1 + 1 = 2，依此类推。

由于这列值可以永远持续下去，定义一个`fibonacci`的求值函数略显困难。显然，返回一整列值并不实际。我们真正需要的，应是某种具有怠惰求值性质的东西——只在必要的时候才进行运算求值。

在Rust中，这样的需求表明，是`Iterator`派上用场的时候了。实现迭代器并不十分困难，但比较繁琐：你得自定义一个类型，弄明白该在其中存储什么，然后为它实现`Iterator` trait。

其实，递推关系足够简单；几乎所有的递推关系都可被抽象出来，变成一小段由宏驱动的代码生成机制。

好了，说得已经足够多了，让我们开始干活吧。

## 构建过程

通常来说，在构建新宏时，我所做的第一件事，是决定宏调用的形式。在我们当前所讨论的情况下，我的初次尝试是这样：

```rust
let fib = recurrence![a[n] = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
```

以此为基点，我们可以向宏的定义迈出第一步，即便在此时我们尚不了解该宏的展开部分究竟是什么样子。此步骤的用处在于，如果在此处无法明确如何解析输入语法，那就可能意味着，整个宏的构思需要改变。

```rust
macro_rules! recurrence {
    ( a[n] = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}
# fn main() {}
```

假装你并不熟悉相应的语法，让我来解释。上述代码块使用`macro_rules!`系统定义了一个宏，称为`recurrence!`。此宏仅包含一条解析规则，它规定，此宏必须依次匹配下列项目：

- 一段字面标记序列，`a` `[` `n` `]` `=`；
- 一段重复 (`$( ... )`)序列，由`,`分隔，允许重复一或多次(`+`)；重复的内容允许：
    - 一个有效的表达式，它将被捕获至变量`inits` (`$inits:expr`)
- 又一段字面标记序列， `...` `,`；
- 一个有效的表达式，将被捕获至变量`recur` (`$recur:expr`)。

最后，规则声明，如果输入被成功匹配，则对该宏的调用将被标记序列`/* ... */`替换。

值得注意的是，`inits`，如它命名采用的复数形式所暗示的，实际上包含所有成功匹配进此重复的表达式，而不仅是第一或最后一个。不仅如此，它们将被捕获成一个序列，而不是——举例说——把它们不可逆地粘贴在一起。还注意到，可用`*`替换`+`来表示允许“0或多个”重复。宏系统并不支持“0或1个”或任何其它更加具体的重复形式。

作为练习，我们将采用上面提及的输入，并研究它被处理的过程。“位置”列将揭示下一个需要被匹配的句法模式，由“⌂”标出。注意在某些情况下下一个可用元素可能存在多个。“输入”将包括所有尚未被消耗的标记。`inits`和`recur`将分别包含其对应绑定的内容。

<style type="text/css">
    /* Customisations. */

    .small-code code {
        font-size: 60%;
    }
    
    table pre.rust {
        margin: 0;
        border: 0;
    }
    
    table.parse-table code {
        white-space: pre-wrap;
        background-color: transparent;
        border: none;
    }
    
    table.parse-table tbody > tr > td:nth-child(1) > code:nth-of-type(2) {
        color: red;
        margin-top: -0.7em;
        margin-bottom: -0.6em;
    }
    
    table.parse-table tbody > tr > td:nth-child(1) > code {
        display: block;
    }
    
    table.parse-table tbody > tr > td:nth-child(2) > code {
        display: block;
    }
</style>

<table class="parse-table">
    <thead>
        <tr>
            <th>位置</th>
            <th>输入</th>
            <th><code>inits</code></th>
            <th><code>recur</code></th>
        </tr>
    </thead>
    <tbody class="small-code">
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>⌂</code></td>
            <td><code>a[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code> ⌂</code></td>
            <td><code>[n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>  ⌂</code></td>
            <td><code>n] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>   ⌂</code></td>
            <td><code>] = 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>     ⌂</code></td>
            <td><code>= 0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>       ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂</code></td>
            <td><code>0, 1, ..., a[n-1] + a[n-2]</code></td>
            <td></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂</code></td>
            <td><code>, 1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>注意</em>：这有两个 ⌂，因为下个输入标记既能匹配 重复元素间的分隔符逗号，也能匹配 标志重复结束的逗号。宏系统将同时追踪这两种可能，直到决定具体选择为止。

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>1, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                     ⌂  ⌂ <s>⌂</s></code></td>
            <td><code>, ..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>注意</em>：第三个被划掉的记号表明，基于上个被消耗的标记，宏系统排除了一项先前存在的可能。

            </td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         ⌂                ⌂</code></td>
            <td><code>..., a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>         <s>⌂</s>                    ⌂</code></td>
            <td><code>, a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                ⌂</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
            <td><code>0</code>, <code>1</code></td>
            <td></td>
        </tr>
        <tr>
            <td><code>a[n] = $($inits:expr),+ , ... , $recur:expr</code>
                <code>                                           ⌂</code></td>
            <td></td>
            <td><code>0</code>, <code>1</code></td>
            <td><code>a[n-1] + a[n-2]</code></td>
        </tr>
        <tr>
            <td colspan="4" style="font-size:.7em;">

<em>注意</em>：这一步表明，类似<tt>$recur:expr</tt>的绑定将消耗<em>一个完整的表达式</em>。此处，究竟什么算是一个完整的表达式，将由编译器决定。稍后我们会谈到语言其它部分的类似行为。

            </td>
        </tr>
    </tbody>
</table>

<p></p>

从此表中得到的最关键收获在于，宏系统会依次尝试将提供给它的每个标记当作输入，与提供给它的每条规则进行匹配。我们稍后还将谈回到这一“尝试”。

接下来我们首先将写出宏调用完全展开后的形态。我们想要的结构类似：

```rust
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }
```

它就是我们实际使用的迭代器类型。其中，`mem`负责存储最近算得的两个斐波那契值，保证递推计算能够顺利进行；`pos`则负责记录当前的`n`值。

> **附注**：此处选用`u64`是因为，对斐波那契数列来说，它已经“足够”了。先不必担心它是否适用于其它数列，我们会提到这一点的。

```rust
    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
```

我们需要这个`if`分支来返回序列的初始值，没什么花哨。

```rust
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val);

                self.pos += 1;
                Some(next_val)
            }
        }
    }
```

这段稍微难办一点。对于具体如何定义`a`，我们稍后再提。`TODO_shuffle_down_and_append`的真面目也将留到稍后揭晓；我们想让它做到：将`next_val`放至数组末尾，并将数组中剩下的元素依次前移一格，最后丢掉原先的首元素。

```rust

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

最后，我们返回一个该结构的实例。在随后的代码中，我们将用它来进行迭代。综上所述，完整的展开应该如下：

```rust
let fib = {
    struct Recurrence {
        mem: [u64; 2],
        pos: usize,
    }

    impl Iterator for Recurrence {
        type Item = u64;

        #[inline]
        fn next(&mut self) -> Option<u64> {
            if self.pos < 2 {
                let next_val = self.mem[self.pos];
                self.pos += 1;
                Some(next_val)
            } else {
                let a = /* something */;
                let n = self.pos;
                let next_val = (a[n-1] + a[n-2]);

                self.mem.TODO_shuffle_down_and_append(next_val.clone());

                self.pos += 1;
                Some(next_val)
            }
        }
    }

    Recurrence { mem: [0, 1], pos: 0 }
};

for e in fib.take(10) { println!("{}", e) }
```

> **附注**：是的，这样做的确意味着每次调用该宏时，我们都会重新定义并实现一个`Recurrence`结构。如果`#[inline]`属性应用得当，在最终编译出的二进制文件中，大部分冗余都将被优化掉。

在写展开部分的过程中时常检查，也是一个有效的技巧。如果在过程中发现，展开中的某些内容需要根据调用的不同发生改变，但这些内容并未被我们的宏语法定义囊括；那就要去考虑，应该怎样去引入它们。在此示例中，我们先前用过一次`u64`，但调用端想要的类型不一定是它；然而我们的宏语法并没有提供其它选择。因此，我们可以做一些修改。

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

/*
let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

for e in fib.take(10) { println!("{}", e) }
*/
# fn main() {}
```

我们加入了一个新的捕获`sty`，它应是一个类型(type)。

> **附注**：如果你不清楚的话，在捕获冒号之后的部分，可是几种语法匹配候选项之一。最常用的包括`item`，`expr`和`ty`。完整的解释可在[宏，彻底解析-`macro_rules!`-捕获](mbe-macro-rules.html#捕获)部分找到。
>
> 还有一点值得注意：为方便语言的未来发展，对于跟在某些特定的匹配之后的标记，编译器施加了一些限制。这种情况常在试图匹配至表达式(expression)或语句(statement)时出现：它们后面仅允许跟进`=>`，`,`和`;`这些标记之一。
>
> 完整清单可在[宏，彻底解析-细枝末节-再探捕获与展开](mbe-min-captures-and-expansion-redux.md)找到。

## 索引与移位

在此节中我们将略去一些实际上与宏的联系不甚紧密的内容。本节我们的目标是，让用户可以通过索引`a`来访问数列中先前的值。`a`应该如同一个切口，让我们得以持续访问数列中最近几个(在本例中，两个)值。

通过采用封装类，我们可以相对简单地做到这点：

```rust
struct IndexOffset<'a> {
    slice: &'a [u64; 2],
    offset: usize,
}

impl<'a> Index<usize> for IndexOffset<'a> {
    type Output = u64;

    #[inline(always)]
    fn index<'b>(&'b self, index: usize) -> &'b u64 {
        use std::num::Wrapping;

        let index = Wrapping(index);
        let offset = Wrapping(self.offset);
        let window = Wrapping(2);

        let real_index = index - offset + window;
        &self.slice[real_index.0]
    }
}
```

> **附注**：对于新接触Rust的人来说，生命周期的概念经常需要一番思考。我们给出一些简单的解释：`'a`和`'b`是生命周期参数，它们被用于记录引用(即一个指向某些数据的借用指针)的有效期。在此例中，`IndexOffset`借用了一个指向我们迭代器数据的引用，因此，它需要记录该引用的有效期，记录者正是`'a`。
>
> 我们用到`'b`，是因为`Index::index`函数(下标句法正是通过此函数实现的)的一个参数也需要生命周期。`'a`和`'b`不一定在所有情况下都相同。我们并没有显式地声明`'a`和`'b`之间有任何联系，但借用检查器(borrow checker)总会确保内存安全性不被意外破坏。

`a`地定义将随之变为：

```rust
let a = IndexOffset { slice: &self.mem, offset: n };
```

如何处理`TODO_shuffle_down_and_append`是我们现在剩下的唯一问题了。我没能在标准库中寻得可以直接使用的方法，但自己造一个出来并不难。

```rust
{
    use std::mem::swap;

    let mut swap_tmp = next_val;
    for i in (0..2).rev() {
        swap(&mut swap_tmp, &mut self.mem[i]);
    }
}
```

它把新值替换至数组末尾，并把其他值向前移动一位。

> **附注**：采用这种做法，将使得我们的代码可同时被用于不可拷贝(non-copyable)的类型。

至此，最终起作用的代码将是：

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => { /* ... */ };
}

fn main() {
    /*
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
    */
    let fib = {
        use std::ops::Index;

        struct Recurrence {
            mem: [u64; 2],
            pos: usize,
        }

        struct IndexOffset<'a> {
            slice: &'a [u64; 2],
            offset: usize,
        }

        impl<'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;

            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;

                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(2);

                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }

        impl Iterator for Recurrence {
            type Item = u64;

            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < 2 {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset { slice: &self.mem, offset: n };
                        (a[n-1] + a[n-2])
                    };

                    {
                        use std::mem::swap;

                        let mut swap_tmp = next_val;
                        for i in (0..2).rev() {
                            swap(&mut swap_tmp, &mut self.mem[i]);
                        }
                    }

                    self.pos += 1;
                    Some(next_val)
                }
            }
        }

        Recurrence { mem: [0, 1], pos: 0 }
    };

    for e in fib.take(10) { println!("{}", e) }
}
```

注意我们改变了`n`与`a`的声明顺序，同时将它们(与递推表达式一同)用一个新区块包裹了起来。改变声明顺序的理由很明显(`n`得在`a`前被定义才能被`a`使用)。而包裹的理由则是：如果不，借用引用`&self.mem`将会阻止随后的`swap`操作(在某物仍存在其它别名时，无法对其进行改变)。包裹区块将确保`&self.mem`产生的借用在彼时过期。

顺带一提，将交换`mem`的代码包进区块里的唯一原因，正是为了缩减`std::mem::swap`的可用范畴，以保持代码整洁。

如果我们直接拿上段代码来跑，将会得到：

```text
0
1
1
2
3
5
8
13
21
34
```

成功了！现在，让我们把这段代码复制粘贴进宏的展开部分，并把它们原本所在的位置换成一次宏调用。这样我们得到：

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ , ... , $recur:expr ) => {
        {
            /*
                What follows here is *literally* the code from before,
                cut and pasted into a new position.  No other changes
                have been made.
            */

            use std::ops::Index;

            struct Recurrence {
                mem: [u64; 2],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [u64; 2],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = u64;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b u64 {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = u64;

                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < 2 {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..2).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [0, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

显然，宏的捕获尚未被用到，但这点很好改。不过，如果尝试编译上述代码，`rustc`会中止，并显示：

```text
recurrence.rs:69:45: 69:48 error: local ambiguity: multiple parsing options: built-in NTs expr ('inits') or 1 other options.
recurrence.rs:69     let fib = recurrence![a[n]: u64 = 0, 1, ..., a[n-1] + a[n-2]];
                                                             ^~~
```

这里我们撞上了`macro_rules`的一处限制。问题出在那第二个逗号上。当在展开过程中遇见它时，编译器无法决定是该将它解析成`inits`中的又一个表达式，还是解析成`...`。很遗憾，它不够聪明，没办法意识到`...`不是一个有效的表达式，所以它选择了放弃。理论上来说，上述代码应该能奏效，但当前它并不能。

> **附注**：有关宏系统如何解读我们的规则，我之前的确撒了点小谎。通常来说，宏系统确实应当如我前述的那般运作，但在这里它没有。`macro_rules`的机制，由此看来，是存在一些小毛病的；我们得记得偶尔去做一些调控，好让它我们期许的那般运作。
>
> 在本例中，问题有两个。其一，宏系统不清楚各式各样的语法元素(如表达式)可由什么样的东西构成，或不能由什么样的东西构成；那是语法解析器的工作。其二，在试图捕获复合语法元素(如表达式)的过程中，它无法不100%地首先陷入该捕获中去。
>
> 换句话说，宏系统可以向语法解析器发出请求，让后者试图把某段输入当作表达式来进行解析；但此间无论语法解析器遇见任何问题，都将中止整个进程以示回应。目前，宏系统处理这种窘境的唯一方式，就是对任何可能产生此类问题的情境加以禁止。
>
> 好的一面在于，对于这摊子情况，没有任何人感到高兴。关键词`macro`早已被预留，以备未来更加严密的宏系统使用。直到那天来临之前，我们还是只得该怎么做就怎么做。

还好，修正方案也很简单：从宏句法中去掉逗号即可。出于平衡考量，我们将移除`...`双边的逗号：[^译注1]

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//                                     ^~~ changed
        /* ... */
#         // Cheat :D
#         (vec![0u64, 1, 2, 3, 5, 8, 13, 21, 34]).into_iter()
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
//                                         ^~~ changed

    for e in fib.take(10) { println!("{}", e) }
}
```

成功！现在，我们该将捕获部分捕获到的内容替代进展开部分中了。

[^译注1]: 在目前的稳定版本(Rust 1.14.0)下，去掉双边逗号后的代码无法通过编译。`rustc`报错“error: `$inits:expr` may be followed by `...`, which is not allowed for `expr` fragments”。解决方案是将这两处逗号替换为其它字面值，如分号；与之前的捕获所用分隔符不同即可。

### 替换

在宏中替换你捕获到的内容相对简单，通过`$sty:ty`捕获到的内容可用`$sty`来替换。好，让我们换掉那些`u64`吧：

```rust
macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            struct Recurrence {
                mem: [$sty; 2],
//                    ^~~~ changed
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; 2],
//                          ^~~~ changed
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;
//                            ^~~~ changed

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
//                                                          ^~~~ changed
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(2);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;
//                          ^~~~ changed

                #[inline]
                fn next(&mut self) -> Option<$sty> {
//                                           ^~~~ changed
                    /* ... */
#                     if self.pos < 2 {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let n = self.pos;
#                             let a = IndexOffset { slice: &self.mem, offset: n };
#                             (a[n-1] + a[n-2])
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..2).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
                }
            }

            Recurrence { mem: [1, 1], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

现在让我们来尝试更难的：如何将`inits`同时转变为字面值`[0, 1]`以及数组类型`[$sty; 2]`。首先我们试试：

```rust
            Recurrence { mem: [$($inits),+], pos: 0 }
//                             ^~~~~~~~~~~ changed
```

此段代码与捕获的效果正好相反：将`inits`捕得的内容排列开来，总共有1或多次，每条内容之间用逗号分隔。展开的结果与期望一致，我们得到标记序列：`0, 1`。

不过，通过`inits`转换出字面值`2`需要一些技巧。没有直接可行的方法，但我们可以通过另一个宏做到。我们一步一步来。

```rust
macro_rules! count_exprs {
    /* ??? */
#     () => {}
}
# fn main() {}
```

先写显而易见的情况：未给表达式时，我们期望`count_exprs`展开为字面值`0`。

```rust
macro_rules! count_exprs {
    () => (0);
//  ^~~~~~~~~~ added
}
# fn main() {
#     const _0: usize = count_exprs!();
#     assert_eq!(_0, 0);
# }
```

> **附注**：你可能已经注意到了，这里的展开部分我用的是括号而非花括号。`macro_rules`其实不关心你用的是什么，只要它成对匹配即可：`( )`，`{ }`或`[ ]`。实际上，宏本身的匹配符(即紧跟宏名称后的匹配符)、语法规则外的匹配符及相应展开部分外的匹配符都可以替换。
>
> 调用宏时的括号也可被替换，但有些限制：当宏被以`{...}`或`(...);`形式调用时，它总是会被解析为一个条目(item，比如，`struct`或`fn`声明)。在函数体内部时，这一特征很重要，它将消除“解析成表达式”和“解析成语句”之间的歧义。

有一个表达式的情况该怎么办？应该展开为字面值`1`。

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
//  ^~~~~~~~~~~~~~~~~ added
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
# }
```

两个呢？

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (2);
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~ added
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

通过递归调用重新表达，我们可将扩展部分“精简”出来：

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
//                           ^~~~~~~~~~~~~~~~~~~~~ changed
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
# }
```

这样做可行是因为，Rust可将`1 + 1`合并成一个常量。那么，三种表达式的情况呢？

```rust
macro_rules! count_exprs {
    () => (0);
    ($e:expr) => (1);
    ($e0:expr, $e1:expr) => (1 + count_exprs!($e1));
    ($e0:expr, $e1:expr, $e2:expr) => (1 + count_exprs!($e1, $e2));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ added
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **附注**：你可能会想，我们是否能翻转这些规则的排列顺序。在此情境下，可以。但在有些情况下，宏系统可能会对此挑剔。如果你发现自己有一个包含多项规则的宏系统老是报错，或给出期望外的结果；但你发誓它应该能用，试着调换一下规则的排序吧。

我们希望你现在已经能看出规律。通过匹配至一个表达式加上0或多个表达式并展开成1+a，我们可以减少规则列表的数目：

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
//  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ changed
}
# fn main() {
#     const _0: usize = count_exprs!();
#     const _1: usize = count_exprs!(x);
#     const _2: usize = count_exprs!(x, y);
#     const _3: usize = count_exprs!(x, y, z);
#     assert_eq!(_0, 0);
#     assert_eq!(_1, 1);
#     assert_eq!(_2, 2);
#     assert_eq!(_3, 3);
# }
```

> **<abbr title="Just for this example">仅对此例</abbr>**：这段代码并非计数仅有或其最好的方法。若有兴趣，稍后可以研读[计数](blk-counting.md)一节。

有此工具后，我们可再次修改`recurrence`，确定`mem`所需的大小。

```rust
// added:
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);
//          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ added

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
//                          ^~~~~~~~ changed
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
//                                ^~~~~~~~ changed
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);
//                                        ^~~~~~~~ changed

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
//                                ^~~~~~~~ changed
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            (a[n-1] + a[n-2])
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
//                                       ^~~~~~~~ changed
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}
/* ... */
# 
# fn main() {
#     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
# 
#     for e in fib.take(10) { println!("{}", e) }
# }
```

完成之后，我们开始替换最后的`recur`表达式。

```rust
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr $(, $tail:expr)*) => (1 + count_exprs!($($tail),*));
# }
# macro_rules! recurrence {
#     ( a[n]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             const MEMORY: uint = count_exprs!($($inits),+);
#             struct Recurrence {
#                 mem: [$sty; MEMORY],
#                 pos: uint,
#             }
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEMORY],
#                 offset: uint,
#             }
#             impl<'a> Index<uint, $sty> for IndexOffset<'a> {
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: &uint) -> &'b $sty {
#                     let real_index = *index - self.offset + MEMORY;
#                     &self.slice[real_index]
#                 }
#             }
#             impl Iterator<u64> for Recurrence {
/* ... */
                #[inline]
                fn next(&mut self) -> Option<u64> {
                    if self.pos < MEMORY {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let n = self.pos;
                            let a = IndexOffset { slice: &self.mem, offset: n };
                            $recur
//                          ^~~~~~ changed
                        };
                        {
                            use std::mem::swap;
                            let mut swap_tmp = next_val;
                            for i in range(0, MEMORY).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }
                        self.pos += 1;
                        Some(next_val)
                    }
                }
/* ... */
#             }
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# fn main() {
#     let fib = recurrence![a[n]: u64 = 1, 1 ... a[n-1] + a[n-2]];
#     for e in fib.take(10) { println!("{}", e) }
# }
```

现在试图编译的话...

```text
recurrence.rs:77:48: 77:49 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:50: 77:51 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                  ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:57: 77:58 error: unresolved name `a`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                         ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
recurrence.rs:77:59: 77:60 error: unresolved name `n`
recurrence.rs:77     let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];
                                                                           ^
recurrence.rs:7:1: 74:2 note: in expansion of recurrence!
recurrence.rs:77:15: 77:64 note: expansion site
```

...等等，什么情况？这没道理...让我们看看宏究竟展开成了什么样子。

```shell
$ rustc -Z unstable-options --pretty expanded recurrence.rs
```

参数`--pretty expanded`将促使`rustc`展开宏，并将输出的AST再重转为源代码。此选项当前被认定为是`unstable`，因此我们还要添加`-Z unstable-options`。输出的信息(经过整理格式后)如下；特别留意`$recur`被替换掉的位置：

```ignore
#![feature(no_std)]
#![no_std]
#[prelude_import]
use std::prelude::v1::*;
#[macro_use]
extern crate std as std;
fn main() {
    let fib = {
        use std::ops::Index;
        const MEM_SIZE: usize = 1 + 1;
        struct Recurrence {
            mem: [u64; MEM_SIZE],
            pos: usize,
        }
        struct IndexOffset<'a> {
            slice: &'a [u64; MEM_SIZE],
            offset: usize,
        }
        impl <'a> Index<usize> for IndexOffset<'a> {
            type Output = u64;
            #[inline(always)]
            fn index<'b>(&'b self, index: usize) -> &'b u64 {
                use std::num::Wrapping;
                let index = Wrapping(index);
                let offset = Wrapping(self.offset);
                let window = Wrapping(MEM_SIZE);
                let real_index = index - offset + window;
                &self.slice[real_index.0]
            }
        }
        impl Iterator for Recurrence {
            type Item = u64;
            #[inline]
            fn next(&mut self) -> Option<u64> {
                if self.pos < MEM_SIZE {
                    let next_val = self.mem[self.pos];
                    self.pos += 1;
                    Some(next_val)
                } else {
                    let next_val = {
                        let n = self.pos;
                        let a = IndexOffset{slice: &self.mem, offset: n,};
                        a[n - 1] + a[n - 2]
                    };
                    {
                        use std::mem::swap;
                        let mut swap_tmp = next_val;
                        {
                            let result =
                                match ::std::iter::IntoIterator::into_iter((0..MEM_SIZE).rev()) {
                                    mut iter => loop {
                                        match ::std::iter::Iterator::next(&mut iter) {
                                            ::std::option::Option::Some(i) => {
                                                swap(&mut swap_tmp, &mut self.mem[i]);
                                            }
                                            ::std::option::Option::None => break,
                                        }
                                    },
                                };
                            result
                        }
                    }
                    self.pos += 1;
                    Some(next_val)
                }
            }
        }
        Recurrence{mem: [0, 1], pos: 0,}
    };
    {
        let result =
            match ::std::iter::IntoIterator::into_iter(fib.take(10)) {
                mut iter => loop {
                    match ::std::iter::Iterator::next(&mut iter) {
                        ::std::option::Option::Some(e) => {
                            ::std::io::_print(::std::fmt::Arguments::new_v1(
                                {
                                    static __STATIC_FMTSTR: &'static [&'static str] = &["", "\n"];
                                    __STATIC_FMTSTR
                                },
                                &match (&e,) {
                                    (__arg0,) => [::std::fmt::ArgumentV1::new(__arg0, ::std::fmt::Display::fmt)],
                                }
                            ))
                        }
                        ::std::option::Option::None => break,
                    }
                },
            };
        result
    }
}
```

呃..这看起来完全合法！如果我们加上几条`#![feature(...)]`属性，并把它送去给一个nightly版本的`rustc`，甚至真能通过编译...究竟什么情况？！

> **附注**：上述代码无法通过非nightly版`rustc`编译。这是因为，`println!`宏的展开结果依赖于编译器内部的细节，这些细节尚未被公开稳定化。

### 保持卫生

这儿的问题在于，Rust宏中的标识符具有卫生性。这就是说，出自不同上下文的标识符不可能发生冲突。作为演示，举个简单的例子。

```rust
# /*
macro_rules! using_a {
    ($e:expr) => {
        {
            let a = 42i;
            $e
        }
    }
}

let four = using_a!(a / 10);
# */
# fn main() {}
```

此宏接受一个表达式，然后把它包进一个定义了变量`a`的区块里。我们随后用它绕个弯子来求`4`。这个例子中实际上存在2种句法上下文，但我们看不见它们。为了帮助说明，我们给每个上下文都上一种不同的颜色。我们从未展开的代码开始上色，此时仅看得见一种上下文：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

现在，展开宏调用。

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

可以看到，在宏中定义的<code><span class="synctx-1">a</span></code>与调用所提供的<code><span class="synctx-0">a</span></code>处于不同的上下文中。因此，虽然它们的字母表示一致，编译器仍将它们视作完全不同的标识符。

宏的这一特性需要格外留意：它们可能会产出无法通过编译的AST；但同样的代码，手写或通过`--pretty expanded`转印出来则能够通过编译。

解决方案是，采用合适的句法上下文来捕获标识符。我们沿用上例，并作修改：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span>:<span class="ident">ident</span>, <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span>, <span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

现在它将展开为：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> </span><span class="synctx-0"><span class="ident">a</span></span><span class="synctx-1"> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

上下文现在匹配了，编译通过。我们的`recurrence!`宏也可被如此调整：显式地捕获`a`与`n`即可。调整后我们得到：

```rust
macro_rules! count_exprs {
    () => (0);
    ($head:expr) => (1);
    ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
}

macro_rules! recurrence {
    ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
//    ^~~~~~~~~~   ^~~~~~~~~~ changed
        {
            use std::ops::Index;

            const MEM_SIZE: usize = count_exprs!($($inits),+);

            struct Recurrence {
                mem: [$sty; MEM_SIZE],
                pos: usize,
            }

            struct IndexOffset<'a> {
                slice: &'a [$sty; MEM_SIZE],
                offset: usize,
            }

            impl<'a> Index<usize> for IndexOffset<'a> {
                type Output = $sty;

                #[inline(always)]
                fn index<'b>(&'b self, index: usize) -> &'b $sty {
                    use std::num::Wrapping;

                    let index = Wrapping(index);
                    let offset = Wrapping(self.offset);
                    let window = Wrapping(MEM_SIZE);

                    let real_index = index - offset + window;
                    &self.slice[real_index.0]
                }
            }

            impl Iterator for Recurrence {
                type Item = $sty;

                #[inline]
                fn next(&mut self) -> Option<$sty> {
                    if self.pos < MEM_SIZE {
                        let next_val = self.mem[self.pos];
                        self.pos += 1;
                        Some(next_val)
                    } else {
                        let next_val = {
                            let $ind = self.pos;
//                              ^~~~ changed
                            let $seq = IndexOffset { slice: &self.mem, offset: $ind };
//                              ^~~~ changed
                            $recur
                        };

                        {
                            use std::mem::swap;

                            let mut swap_tmp = next_val;
                            for i in (0..MEM_SIZE).rev() {
                                swap(&mut swap_tmp, &mut self.mem[i]);
                            }
                        }

                        self.pos += 1;
                        Some(next_val)
                    }
                }
            }

            Recurrence { mem: [$($inits),+], pos: 0 }
        }
    };
}

fn main() {
    let fib = recurrence![a[n]: u64 = 0, 1 ... a[n-1] + a[n-2]];

    for e in fib.take(10) { println!("{}", e) }
}
```

通过编译了！接下来，我们试试别的数列。

```rust
# macro_rules! count_exprs {
#     () => (0);
#     ($head:expr) => (1);
#     ($head:expr, $($tail:expr),*) => (1 + count_exprs!($($tail),*));
# }
# 
# macro_rules! recurrence {
#     ( $seq:ident [ $ind:ident ]: $sty:ty = $($inits:expr),+ ... $recur:expr ) => {
#         {
#             use std::ops::Index;
#             
#             const MEM_SIZE: usize = count_exprs!($($inits),+);
#     
#             struct Recurrence {
#                 mem: [$sty; MEM_SIZE],
#                 pos: usize,
#             }
#     
#             struct IndexOffset<'a> {
#                 slice: &'a [$sty; MEM_SIZE],
#                 offset: usize,
#             }
#     
#             impl<'a> Index<usize> for IndexOffset<'a> {
#                 type Output = $sty;
#     
#                 #[inline(always)]
#                 fn index<'b>(&'b self, index: usize) -> &'b $sty {
#                     use std::num::Wrapping;
#                     
#                     let index = Wrapping(index);
#                     let offset = Wrapping(self.offset);
#                     let window = Wrapping(MEM_SIZE);
#                     
#                     let real_index = index - offset + window;
#                     &self.slice[real_index.0]
#                 }
#             }
#     
#             impl Iterator for Recurrence {
#                 type Item = $sty;
#     
#                 #[inline]
#                 fn next(&mut self) -> Option<$sty> {
#                     if self.pos < MEM_SIZE {
#                         let next_val = self.mem[self.pos];
#                         self.pos += 1;
#                         Some(next_val)
#                     } else {
#                         let next_val = {
#                             let $ind = self.pos;
#                             let $seq = IndexOffset { slice: &self.mem, offset: $ind };
#                             $recur
#                         };
#     
#                         {
#                             use std::mem::swap;
#     
#                             let mut swap_tmp = next_val;
#                             for i in (0..MEM_SIZE).rev() {
#                                 swap(&mut swap_tmp, &mut self.mem[i]);
#                             }
#                         }
#     
#                         self.pos += 1;
#                         Some(next_val)
#                     }
#                 }
#             }
#     
#             Recurrence { mem: [$($inits),+], pos: 0 }
#         }
#     };
# }
# 
# fn main() {
for e in recurrence!(f[i]: f64 = 1.0 ... f[i-1] * i as f64).take(10) {
    println!("{}", e)
}
# }
```

运行上述代码得到：

```text
1
1
2
6
24
120
720
5040
40320
362880
```

成功了！
