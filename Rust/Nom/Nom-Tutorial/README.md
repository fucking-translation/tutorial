# nom5 教程

[原文](https://github.com/benkay86/nom-tutorial)

[nom](https://github.com/Geal/nom)是一个用 Rust 编写的解析器组合器库。它可以处理二进制和文本文件。在需要使用正则表达式，Flex或Bison的地方可以考虑使用它。nom 具有 Rust 的强类型和内存安全的优势，它通常比其他可替代的工具性能更好。学习 nom 是 Rust 工具箱中的一项有价值的补充。

## 基本原理

nom 的官方文档包含一些简单的示例(如：如何解析十六进制RGB颜色代码)以及非常复杂的示例(如：如何解析json)。当我第一次学习 nom 时，我在简单示例和复杂示例之间发现了陡峭的学习曲线。此外，nom 的早期版本和大多数现有文档都使用宏。从 nom 5.0 开始，不赞成使用宏，而推荐使用函数。本教程旨在通过解析`/proc/mounts`的内容来填补简单解析器和复杂解析器之间的空白，并演示使用函数替代宏的方法。

## Table of Contents

1. [mount 命令回顾](#chap1)
2. [环境搭建](#chap2)
3. [你好，解析器](#chap3)
4. [阅读 nom 文档](#chap4)
5. [奠定基础](#chap5)
6. [它不是空格](#chap6)
7. [最棒的转义](#chap7)
8. [挂载选项](#chap8)
9. [整合全部内容](#chap9)
10. [迭代器是点睛之笔](#chap10)

## <a name="chap1"></a> 挂载命令回顾

如果你使用的是 Linux 系统，你可能对`mount`命令很熟悉了。如果你运行`mount`却不带任何参数，它将会在终端上打印出已挂载文件系统的清单。

```console
$ mount
sysfs on /sys type sysfs (rw,seclabel,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
...output trimmed for length...
```

使用 Rust 实现`mount`命令的全部功能超出了本教程的范围，但是我们可以借助 nom 实现上面的输出内容。Linux 内核在`proc/mounts`中存储了当前所有关于已挂载文件系统的信息。

```console
$ cat /proc/mounts
sysfs /sys sysfs rw,seclabel,nosuid,nodev,noexec,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
```

每个挂载都在单独的一行中描述。在每一行中，挂载的属性使用空格分开。

1. 设备 (如：sysfs, /dev/sda1)
2. 挂载点 (如：/sys, /mnt/disk)
3. 文件系统类型 (如：sysfs, ext4)
4. 挂载选项，一个用逗号分隔的字符串 (如：rw, ro)
5. 每一行以`0 0`结尾，以模仿(mimic)`/etc/fstab`的格式。第五列的`0 0`只是一个装饰 -- 每行都相同，因此不包含有用的信息。

__在本教程中__，我们会写一个程序将`/proc/mounts`的每一行解析成 Rust 的结构体并将它们打印在控制台上，就像`mount`命令一样。


## <a name="chap2"></a> 环境搭建

为了学习示例的代码，你需要先[安装 Rust](https://www.rust-lang.org/learn/get-started)，并且我假设你已经有了 Rust 语言的基础。然后下载并运行本教程的完整代码：

```console
$ git clone https://github.com/benkay86/nom-tutorial.git
$ cd nom-tutorial
$ cargo run
sysfs on /sys type sysfs (rw,seclabel,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
...output trimmed for length...
```

本教程最终版本的代码需要花费大量时间消化，所以在接下来的章节我们将会一步步的构建。我建议使用`cargo new my-nom-tutorial`来创建自己的 Rust 项目，并保证完整教程的副本作为参考。

```toml
[dependencies]
nom = "5.0"
```

## <a name="chap3"></a> 你好，解析器

在你的新项目中，根据下面的代码编辑`main.rs`的内容：

```rust
extern crate nom;

fn hello_parser(i: &str) -> nom::IResult<&str, &str> {
	nom::bytes::complete::tag("hello")(i)
}

fn main() {
	println!("{:?}", hello_parser("hello"));
	println!("{:?}", hello_parser("hello world"));
	println!("{:?}", hello_parser("goodbye hello again"));
}
```

编译并运行该程序：

```console
$ cargo run
Ok(("", "hello"))
Ok((" world", "hello"))
Err(Error(("goodbye hello again", Tag)))
```

让我们逐行细分该程序。

### 使用 nom crate

```rust
extern crate nom;
```

在[上一节](#chap2)中，我们在`Cargo.toml`添加了 nom 依赖。这一行告诉你的程序关于 nom 库的信息，允许你通过`nom::`来访问这个库。你可以添加诸如`use nom::IResult;`之类的行以减少输入，但是我在本教程中故意使用了冗长的符号，以便你可以清楚的看到模块的层次结构(hierarchy)。

### 创建一个自定义的解析器

```rust
fn hello_parser(i: &str) -> nom::IResult<&str, &str> {
	nom::bytes::complete::tag("hello")(i)
}
```

上述代码创建了一个名为`hello_parser`的函数，该函数将`&str`(被借用的字符串切片)作为输入，并将`nom::IResult<&str, &str>`作为返回类型，我们将在后面对其进行详细的介绍。在函数体中，我们创建了 nom 的 tag 解析器。tag 解析器可以识别文本字符串或者文本的“标签”。 tag 解析器`tag("hello")`是一个函数对象，可以识别文本中的“hello”。我们接着调用 tag 解析器，将输入的字符串作为它的参数并返回结果(请记住，在 Rust 中，你可以在函数的最后一行省略`return`关键字)。

### 调用解析器

```rust
println!("{:?}", hello_parser("hello world"));
// Ok((" world", "hello"))
```

现在让我们来到`main`函数并查看解析器都做了什么。回顾(recall)`println!("{:?}", x)`，打印`x`的调试输出，给我们提供了一种简单的方式来查看 Rust 变量的内容。在这里我们使用几种不同的测试字符串来调用`hello_parser()`，并打印返回的`nom::IResult<&str, &str>`结果。若你所示，它表明`IResult`是一种 Rust 的`Result`，可以包含`Ok`或者`Err`。当解析器执行成功时，将返回其通用类型参数的元组，在本例中为`&str`。元组的第二个元素是解析器的“输出”，通常是解析器匹配或消耗的字符串，在本例中是“hello”。元组的第一个参数是剩下的输入部分。在本例中是“ world”(注意：包含前面的一个空格)。

```rust
println!("{:?}", hello_parser("hello"));
// Ok(("", "hello"))
```

在本例中，tag 消耗来整个输入，因此元组的第一个元素(输入的剩余部分)是一个空字符串。

```rust
println!("{:?}", hello_parser("goodbye hello again"));
// Err(Error(("goodbye hello again", Tag)))
```

在这里，tag 将会返回一个`Err`，因为输入字符串不是以“hello”开头的。注意到即使单词“hello”出现在输入的中间部分，解析器解析依然失败了 -- 大多数 nom 解析器(包括 tag) 将只会匹配输入的开头。`Error`对象是一个`nom::Err::Error((&str, nom::error::ErrorKind))`，他是剩余的输入(解析失败，所以整个输入会完全保留)以及用来描述解析失败的`ErrorKind`组成的元组。你可以在 Github 中阅读更多关于 [nom 错误处理](https://github.com/Geal/nom/blob/master/doc/error_management.md)的内容。

### 总结

* nom解析器通常采用输入`&str`并返回`IResult<&str,&str>`。
* 你可以通过定义`fn (&str) -> IResult<&str,&str>`来组成自己的解析器并返回 nom 解析器某种组合的结果。
* 当解析器成功匹配到一些或全部的输入，它将返回`Ok`，其包含剩余的输入以及消耗的输入组成的元组。
* 当解析器匹配失败将返回一个`Err`。
* 大多数 nom 解析器只匹配输入的开头，即使有模式可以匹配到之后的输入。

## <a name="chap4"></a>阅读 nom 文档

你可能需要经常参考 nom 的[文档](https://docs.rs/nom)。请确保你阅读的是 5.0 以及之后的版本，因为在 nom 4 版本之后有一个较大的改动。之前版本的 nom 是以宏为中心，因此你可以发现很多对宏的引用，如`tag!()`。nom 中的宏已经被逐渐弃用，以支持函数。大多数函数的名称与宏的名称相同，只是没有感叹号，如：`tag()`。你可以在[这里](https://docs.rs/nom/5.0.0/nom/all.html#Functions)查看 nom 中的所有函数清单。

你会发现其中有`streaming`和`complete`子模块。在高级使用中，nom 支持流输入或缓冲输入，这是一种解析器可能会遇到的不完整输入片段。在本教程中，我们将重点关注用于非流输入的`complete`子模块。

* `nom::branch`中的解析器可以对于多个子解析器执行逻辑操作。举个例子：如果有任何一个子解析器执行成功，则`nom::branch::alt`执行成功。
* `nom::bytes::complete`中的解析器可以对字节数组进行操作。我们的朋友`tag`解析器就是属于这个子模块的。
* `nom::character::complete`中解析器可以识别字符。举个例子：`nom::character::complete::multispace1`匹配1或多个空格。
* `nom::combinator`中的解析器允许我们构建解析器的组合。举个例子：`nom::combinator::map`将第一个解析器的输出传递给第二个解析器。
* `nom::multi`中的解析器可以返回输出的集合。举个例子：`nom::multi::separated_list`返回被分隔符分开的字符串向量。
* `nom::number::complete`中的解析器可以匹配数值。
* `nom::sequence`中的解析器可以匹配有限的输入序列。举个例子：`nom::sequence::tuple`接受一个子解析器元组并返回它们输出的元组。

## <a name="chap5"></a> 奠定基础

本节处理设置程序的 non-nom (读出负载难道很无趣吗？)部分。如果你对 Rust 已经十分熟悉并且只是希望阅读关于 nom 部分的内容，你可以直接跳到[下一节](#chap6)。

### 封装

将你的整个程序写在一个文件中很简单，也很迷人。然而，更好的做法是将你的程序分为类库和二进制文件，让基础逻辑更易于重用。在本教程中，我们将使用一种更得体(take the high road)的方式，在`main.rs`的同级目录下创建一个名为`lib.rs`的空文件。Cargo自动知道将`lib.rs`构建到我们在`Cargo.toml`中使用`name="nom-example"`行指定名称“nom-example”的库/crate中。然后让一个新的`main.rs`使用我们的`nom-example`库而不是直接使用 nom。

```rust.rs
extern crate nom_example;

fn main() {
}
```

请注意：当 crate 的名字包含中划线时，我们在 Rust 代码中将其转化为下划线。

### 错误处理

很遗憾，很多 Rust 教程通过书写`could_fail.unwrap()`或`could_fail.expect("Oh no!")`来处理潜在的错误。当这些语句发生错误时，会让你的代码发生 panic。在一个简单的示例中，这一切都运行的很好，但是你应该避免书写会引起 panic 的生产代码。因此我们引入被称为`?`运算符的`could_fail?`语法。这需要一些管道。

```rust
// lib.rs

/// Type-erased errors.
pub type BoxError = std::boxed::Box<dyn
	std::error::Error   // must implement Error to satisfy ?
	+ std::marker::Send // needed for threads
	+ std::marker::Sync // needed for threads
>;
```
```rust
// main.rs

extern crate nom_example;
use nom_example::BoxError;

fn main() -> std::result::Result<(), BoxError> {
	// Inside the body of main we can now use the ? operator.
	Ok(())
}

```


我们的`main()`函数现在返回一个`Result`，其中的错误类型是我们在`lib.rs`中定义的`BoxError`。在`main()`函数的结尾，我们返回带有空元组的`Ok(())`，以表示程序已经正常完成。当我们以这种方式编写`main()`函数或其他函数时，它允许我们编写`could_fail?`，其行为与`could_fail.unwrap()`类似，不同之处在于它在堆栈中返回一个错误而不是 panic。如果你对这个语法不是很熟悉，请参阅[rust程序设计](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html)。

这个神秘的`BoxError`到底是什么？它就是所谓的 [trait 对象](https://doc.rust-lang.org/reference/types/trait-object.html)，在这里，允许你将所有实现了标准[`Error`](https://doc.rust-lang.org/std/error/trait.Error.html)特征的错误传递给调用栈。注意到包含了 [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html) 和 [`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html) 来保证错误是线程安全的；尽管它们的用处现在还不明显，但是当你与并发的代码或类库进行交互时，它就变得非常重要。请参阅 [rust-error-tutorial](https://benkay86.github.io/rust-error-tutorial.html) 来学习它的设计模式。

> 注意：在本教程之前的版本中，我演示了如何编写用于封装 nom 错误的自定义错误类型。[自从 nom 5.1.1 的错误实现了`Error`](https://github.com/Geal/nom/pull/1043)，因此可以与 Rust 中的`?`一起使用。太棒了！

### 存储 Mount 信息

当我们解析`/proc/mounts`中的一行时，我们需要将它解析成某一个结构。让我们在`lib.rs`中添加一个简单的结构体，以存储挂载的信息。请注意：我们可以使用一个 [HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) 存储挂载的选项，但是为了简单起见，我们使用向量。

```rust
#[derive(Clone, Default, Debug)]
pub struct Mount {
	pub device: std::string::String,
	pub mount_point: std::string::String,
	pub file_system_type: std::string::String,
	pub options: std::vec::Vec<std::string::String>,
}
```

## <a name="chap6"></a>它不是空格

使用 nom 构建解析器非常类似于使用乐高建造一样。你从建造最小的零件开始，然后逐步将零件组合在一起，直到获得看起来很酷的城堡或太空飞船。你会回想起`/proc/mounts`中的每一行都是用空格分隔的：

```console
sysfs /sys sysfs rw,seclabel,nosuid,nodev,noexec,relatime 0 0
```

这意味着该行中的每一项都是一个字符或字节序列，而不是空格。我们将从构建一个 nom 解析器开始，该解析器可以识别一个或多个非空格的字节序列。

```rust
pub(self) mod parsers {
	use super::Mount;

	fn not_whitespace(i: &str) -> nom::IResult<&str, &str> {
		nom::bytes::complete::is_not(" \t")(i)
	}
	
	#[cfg(test)]
	mod tests {
		use super::*;
		
		#[test]
		fn test_not_whitespace() {
			assert_eq!(not_whitespace("abcd efg"), Ok((" efg", "abcd")));
			assert_eq!(not_whitespace("abcd\tefg"), Ok(("\tefg", "abcd")));
			assert_eq!(not_whitespace(" abcdefg"), Err(nom::Err::Error((" abcdefg", nom::error::ErrorKind::IsNot))));
		}
	}
}
```

这个解析器的核心是`nom::bytes::complete::is_not(" \t")`， 它是一个 nom 解析器，可以识别一个或多个不是空格或制表符的字节 -- 如：不是空格，正是我们想要的。如果你对创建自定义的解析器(这里被称为`not_whitespace`)的语法不是很熟悉的话，可以回到[你好，解析器](#chap3)的示例中进行回顾。

### 组织

尽管并非一定要使程序正常工作，但我们还是尝试通过封装对良好的编码进行建模。我们将在名为`parsers`的子模块下放置我们所有的 nom 解析器。该子模块是`pub(self)`的，意味着在`lib.rs`中的其他函数可以使用它，但是它不会暴露在我们的crate之外。

我们稍后编写的解析器之一将要使用我们在[上一节](#chap5)定义的`Mount`结构。我们使用`use super::Mount`来使`Mount`结构定义在父模块中，或使`parsers`模块的“super”作用域在`parsers`模块内可见。

### 单元测试

我们还为另一个良好的编程习惯建模，即单元测试。在`parsers`模块中我们定义另一个被称为`tests`的子模块(你可以对其任意命名)。`#cfg[(test)]`行告诉 Cargo `tests`模块应当仅在运行`cargo test`时进行编译。实际的测试在函数`fn test_not_whitespace()`内部进行(take place)。`#[test]`仅放置在函数名之前，它告诉 Cargo 当调用`cargo test`时，它会运行该函数作为单元测试。

在这里，panic 是被允许的。如果没有 panic 则表示单元测试成功。宏`assert_eq!()`将会发生 panic 如果两个参数不相等。我们测试了一些断言(assertion)，在断言中`not_whitespace`解析器应该执行成功并确保空格和每一个输入序列的后续字符不会被消耗。我们也测试了一种情况使解析器应该解析失败。尽管我们的程序还没有完成，你已经可以编译它并确保`not_whitespace`解析器可以和你预期的一样正常运行。

```console
$ cargo test
   Compiling nom-tutorial v0.1.0 (/home/benjamin/nom-tutorial)
    Finished dev [unoptimized + debuginfo] target(s) in 11.11s
     Running target/debug/deps/nom_tutorial-111f8746083b8c53

running 1 tests
test parsers::tests::test_not_whitespace ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out

     Running target/debug/deps/nom_tutorial-a3501c35106b411e

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

## <a name="chap7"></a>最棒的转义

当我们挂载了一个带有空格的目录会发生什么？如果你具有 root 用户的访问权限，可以尝试以下操作，否则请相信我的话，准没错。

```console
$ mkdir "Marry had"
$ mkdir "a little lamb"
$ sudo mount -o bind "a little lamb" "Mary had"
$ cat /proc/mounts
/dev/nvme0n1p3 /home/benjamin/Mary\040had btrfs rw,seclabel,noatime,nodiratime,ssd,discard,space_cache,subvolid=258,subvol=/home/benjamin/a\040little\040lamb 0 0
...output trimmed for length...
```

如你所见，每一个空格可以被`\040`替代。这是许多语言中你可能必须解析的特性，被称为转义。字符`\`是转义符，`040`是转义序列。有时候你可能想要一个`\`，这时你需要使用`\\`对其进行转义。

幸运的是，nom已经有了一个内置的解析器`nom::bytes::complete::escaped_transform`来处理这些转义序列。就像它的名字一样，它将字符数组每一个转义序列转换成字节数组的文本序列。

```rust
pub(self) mod parsers {
	// ...
	
	fn escaped_space(i: &str) -> nom::IResult<&str, &str> {
		nom::combinator::value(" ", nom::bytes::complete::tag("040"))(i)
	}
	
	fn escaped_backslash(i: &str) -> nom::IResult<&str, &str> {
		nom::combinator::recognize(nom::character::complete::char('\\'))(i)
	}
	
	fn transform_escaped(i: &str) -> nom::IResult<&str, std::string::String> {
		nom::bytes::complete::escaped_transform(nom::bytes::complete::is_not("\\"), '\\', nom::branch::alt((escaped_backslash, escaped_space)))(i)	
	}
	
	#[cfg(test)]
	mod tests {
		// ...
		
		#[test]
		fn test_escaped_space() {
			assert_eq!(escaped_space("040"), Ok(("", " ")));
			assert_eq!(escaped_space(" "), Err(nom::Err::Error((" ", nom::error::ErrorKind::Tag))));
		}
		
		#[test]
		fn test_escaped_backslash() {
			assert_eq!(escaped_backslash("\\"), Ok(("", "\\")));
			assert_eq!(escaped_backslash("not a backslash"), Err(nom::Err::Error(("not a backslash", nom::error::ErrorKind::Char))));
		}
		
		#[test]
		fn test_transform_escaped() {
			assert_eq!(transform_escaped("abc\\040def\\\\g\\040h"), Ok(("", std::string::String::from("abc def\\g h"))));
			assert_eq!(transform_escaped("\\bad"), Err(nom::Err::Error(("bad", nom::error::ErrorKind::Tag))));
		}
	}
}
```

### 从简单开始

我们从定义可以识别转义序列(`040`和`\`)的自定义`escaped_space`和`escaped_backslash`解析器开始，并分别(respectively)返回未转义的序列。

`escaped_space`解析器使用`nom::combinator::value`，当子解析器(在本例中是熟悉的`tag`)解析成功后，它返回指定的值(在本例中是一个空格)。我们可以使用这种方式来编写：

```rust
fn escaped_space(i: &str) -> nom::IResult<&str, &str> {
	match nom::bytes::complete::tag("040")(i) {
		Ok((remaining_input, _)) => Ok((remaining_input, " ")),
		Err(e) => Err(e)
	}
}
```

但是 nom 给我们提供了许多方便的解析器，如：`combinator::value`，开箱即用(out-of-the-box)，使用起来更加简单。

### 组合解析器

随着编写并测试了更加简单的子解析器，现在使用`escaped_transform`解析器已经十分简单。如果我们只转义`\040`而不关心`\\`，我们可以这样写：

```rust
nom::bytes::complete::escaped_transform(nom::bytes::complete::is_not("\\"), '\\', escaped_space)(i)	
```

`escaped_transform`需要两个解析器已经一个`字符`作为参数：

1. 不能转义的字节序列。在我们的示例中，我们可以使用熟悉的`bytes::complete::is_not`解析器来匹配一个或多个不是空格字符的字节。
2. `\`是转义字符。
3. 解析器将转义序列转换成了最终的形式(去掉了前面的`\`)。

在我们的示例中，我们需要处理许多转义序列，因此我们使用`nom::branch::alt`，它可以将解析器元组作为参数并当其中一个解析器匹配上的时候返回结果：

```rust
escaped_transform(..., alt((escaped_backslash, escaped_space)))
```

### 返回值类型

到目前为止，我们看到 nom 解析器返回的是一个`IResult<&str, &str>`，但是 nom 解析器仅仅是一个 Rust 函数，它们可以返回任何值。如果你仔细的研究了示例代码你会发现：

```rust
fn transform_escaped(i: &str) -> nom::IResult<&str, std::string::String>
```

这是因为`escaped_transform`解析器不进行内存的拷贝/分配就无法生成自己的输出字符串，因此它返回的是`nom::IResult<&str, std::string::String>`而不是`&str`。

## <a name="chap8"></a>挂载选项

我们已经快到(终点)了。我们必须自定义一个或多个解析器，然后才可以将所有自定义解析器组合到一架灿烂(glorious)的乐高飞船中。下面的自定义解析器将以逗号为分隔符的挂载选项清单(如：`ro,user`)解析成字符串向量(如：`["ro", "user"]`)。到目前为止，你应该已经十分清楚该代码的作用以及它是如何工作的。

```rust
pub(self) mod parsers {
	// ...
	
	fn mount_opts(i: &str) -> nom::IResult<&str, std::vec::Vec<std::string::String>> {
		nom::multi::separated_list(
			nom::character::complete::char(','),
			nom::combinator::map_parser(
				nom::bytes::complete::is_not(", \t"),
				transform_escaped
			)
		)(i)
	}
	
	#[cfg(test)]
	mod tests {
		// ...
		
		#[test]
		fn test_mount_opts() {
			assert_eq!(mount_opts("a,bc,d\\040e"), Ok(("", vec!["a".to_string(), "bc".to_string(), "d e".to_string()])));
		}
	}
}
```
从返回的`mount_opts`类型可以看出，我们将像我们承诺的那样生成一个`Vec<String>`。`multi::separated_list`就是用来做这个的，将某些解析器解析后分隔的列表元素(可能与其他解析器相匹配)转化成一个向量。

1. 列表由`character::complete::char(',')`分隔。
2. 列表的元素必须不能包含逗号。它们也不允许包含空格，因为列表会因为空格而被终止。
3. 我们先使用`combinator::map_parser`在`is_not(", \t")`的输出上调用`transform_escaped`解析器，然后再将其添加到向量中。这让我们可以十分方便的处理转义字符。

## <a name="chap9"></a>整合全部内容

本教程可能看起来有很多的代码并看不到尽头。既然我们已经定义了所有我们需要的解析器，我们将再写一个解析器将所有东西整合到一起。希望当你看到由简单解析器组成高级解析器是这么的简单时，你会感叹道：当使用 nom 时，你的程序是多么的强大。

### 最终的解析器

```rust
pub(self) mod parsers {
	// ...
	
	pub fn parse_line(i: &str) -> nom::IResult<&str, Mount> {
		match nom::combinator::all_consuming(nom::sequence::tuple((
			/* part 1 */
			nom::combinator::map_parser(not_whitespace, transform_escaped), // device
			nom::character::complete::space1,
			nom::combinator::map_parser(not_whitespace, transform_escaped), // mount_point
			nom::character::complete::space1,
			not_whitespace, // file_system_type
			nom::character::complete::space1,
			mount_opts, // options
			nom::character::complete::space1,
			nom::character::complete::char('0'),
			nom::character::complete::space1,
			nom::character::complete::char('0'),
			nom::character::complete::space0,
		)))(i) {
				/* part 2 */
				Ok((remaining_input, (
				device,
				_, // whitespace
				mount_point,
				_, // whitespace
				file_system_type,
				_, // whitespace
				options,
				_, // whitespace
				_, // 0
				_, // whitespace
				_, // 0
				_, // optional whitespace
			))) => {
				/* part 3 */
				Ok((remaining_input, Mount { 
					device: device,
					mount_point: mount_point,
					file_system_type: file_system_type.to_string(),
					options: options
				}))
			}
			Err(e) => Err(e)
		}
	}
	
	#[cfg(test)]
	mod tests {
		// ...
		
		#[test]
		fn test_parse_line() {
			let mount1 = Mount{
				device: "device".to_string(),
				mount_point: "mount_point".to_string(),
				file_system_type: "file_system_type".to_string(),
				options: vec!["options".to_string(), "a".to_string(), "b=c".to_string(), "d e".to_string()]
			};
			let (_, mount2) = parse_line("device mount_point file_system_type options,a,b=c,d\\040e 0 0").unwrap();
			assert_eq!(mount1.device, mount2.device);
			assert_eq!(mount1.mount_point, mount2.mount_point);
			assert_eq!(mount1.file_system_type, mount2.file_system_type);
			assert_eq!(mount1.options, mount2.options);
		}
	
	}
}
```

哇哦，代码有点多！从整体上来看，我们注意到`parse_line`返回了`Mount`结构。我们还注意到它是`pub`的，因为我们希望在`parsers`模块外部调用这个解析器。我们将具体的细节分为三个部分(在代码中用注释标记)：

1. 目前先忽略`all_consuming`解析器，`sequence::tuple`依次匹配一个子解析器元组。在第一部分中，我们提交了一个我们想要匹配的子解析器列表(作为一个元组)。它可以告诉 nom `/proc/mounts`中的每一行的结构：首先是一些非空格字符，后面跟一些空格，然后是一些更多的非空格，后面跟更多的空格，后面有时会跟一些挂载参数，以此类推(so forth)。请注意：我们在调用中是如何溜进带有`transform_escaped`的`map_parser`解析器来处理转义字符的。

2. `sequence::tuple`解析器返回了一个元组，该元组中的每个元素对应它的每一个子解析器。在第二部分中，我们将元组解构成带有描述性名称的局部变量。举个例子：每一行中的第一个非空序列表示设备，因此我们将元组解构后，将其第一个元素命名为`device`。我们使用`_`作为占位符来忽略元素中我们并不关心的元素(如：空格)。
3. 我们使用第二部分解构的局部变量创建并返回一个新的`Mount`对象。

最后，如果这里有剩余的输入，`all_consuming`解析器将会解析失败。如果一行的结尾没有我们所期望的，将会导致`parse_line`保守的(conservatively)返回一个错误。

### 最终解析器的替代方案

我收到了一些有效的反馈说上面的最终解析器太过复杂，无法查看。下面是最终解析器的替代方案，它使用更少的，可读性更强(取决于你的敏感度)的代码来实现相同的功能。它使用了大量的`?`将元组解析器分解成了单独的语句。如果解析失败，`?`操作符会尽早的结束函数并返回一个错误。来自每个解析器的剩余输入会当作下一个解析器的输入。相关的(Pertinent)变量将被存储，之后会在函数的末尾用于构造`Mount`对象，多余的(Superfluous)变量通过分配给`_`来丢弃。

```rust
pub fn parse_line_alternate(i: &str) -> nom::IResult<&str, Mount> {
	let (i, device) = nom::combinator::map_parser(not_whitespace, transform_escaped)(i)?; // device
	let (i, _) = nom::character::complete::space1(i)?;
	let (i, mount_point) = nom::combinator::map_parser(not_whitespace, transform_escaped)(i)?; // mount_point
	let (i, _) = nom::character::complete::space1(i)?;
	let (i, file_system_type) = not_whitespace(i)?; // file_system_type
	let (i, _) = nom::character::complete::space1(i)?;
	let (i, options) = mount_opts(i)?; // options
	let (i, _) = nom::combinator::all_consuming(nom::sequence::tuple((
		nom::character::complete::space1,
		nom::character::complete::char('0'),
		nom::character::complete::space1,
		nom::character::complete::char('0'),
		nom::character::complete::space0
	)))(i)?;
	Ok((i, Mount {
		device: device,
		mount_point: mount_point,
		file_system_type: file_system_type.to_string(),
		options:options
	}))
}
```

你可以尝试一下通过注释掉原始函数并将`parse_line_alternate`命名为`parse_line`。在你的代码中可以使用你更喜欢的风格。

### 测试

你已经可以通过执行`cargo test`来验证程序是否可以正常工作，但是我们可以做的更好一点，让它可以调用我们的可执行文件并一行行的展示挂载列表。我们将定义一个名为`nom_tutorial::mounts()`的函数来将它们打印出来，并可以在`main.rs`中对其进行调用。

`lib.rs`

```rust
// Needed to use traits associated with std::io::BufReader.
use std::io::BufRead;
use std::io::Read;

pub fn mounts() -> Result<(), BoxError> {
	let file = std::fs::File::open("/proc/mounts")?;
	let buf_reader = std::io::BufReader::new(file);
	for line in buf_reader.lines() {
		match parsers::parse_line(&line?[..]) {
			Ok( (_, m) ) => {
				println!("{}", m);
			},
			Err(_) => return Err(ParseError::default().into())
		}
	}
	Ok(())
}
```
`main.rs`

```rust
extern crate nom_tutorial;

fn main() -> std::result::Result<(), BoxError> {
	nom_tutorial::mounts()?
	Ok(())
}
```

我们打开`/proc/mounts`文件，创建一个`BufReader`来一行行的读取输入，然后解析每一行。如果解析失败，我们将其转换成[在之前定义的](#chap5)自定义错误类型`ParseError`。如果解析成功则将`Mount`的选项在新的一行打印出来。可以如下进行尝试：

```console
$ cargo run
/dev/nvme0n1p3 on /home/benjamin/Mary had type btrfs (rw,seclabel,noatime,nodiratime,ssd,discard,space_cache,subvolid=258,subvol=/home/benjamin/a little lamb)
...output trimmed for length...
```

我们可以读取`/proc/mounts`的完整内容并使用`nom::character::complete::line_ending`来修改我们的解析器以识别行结束标识。然而，如果`/proc/mounts`的内容特别长怎么办呢？也许我们正在一个具有数百个已挂载文件系统的大型服务器上工作，这会导致`/proc/mounts`的内容达到上百兆字节！(好吧，。这可能在现实生活中不会出现) 由于 Rust 已经为我们提供了另一种解析行尾的方式(`BufReader`)，因此我们不妨利用它来降低(理论上的)内存使用量并使我们的解析器足够简单。


## <a name="chap10"></a>迭代器是点睛之笔

从将我们的解析器分解成类库和二进制的角度(standpoint)来看，仅仅具有一个可以打印出挂载列表的函数不太符合人体工程学(ergonomic)。本教程的最终版本，你可以从 Github 上下载，引入了一个类型为`Mounts`的新对象，它在内部管理`/proc/mounts`上的`BufReader`并实现了`IntoIterator`接口。这让我们在`main.rs`中可以这样编写：

```rust
extern crate nom_tutorial;

fn main() -> std::result::Result<(), BoxError> {
	for mount in nom_tutorial::mounts()? {
		println!("{}", mount?);
	}
	Ok(())
}
```

想要知道这到底有多强大，我们可以尝试一下：

```rust 
extern crate nom_tutorial;

fn main() -> std::result::Result<(), BoxError> {
	for mount in nom_tutorial::mounts()? {
		let mount = mount?; // Result --> Mount
		println!("The device \"{}\" is mounted at \"{}\".", mount.device, mount.mount_point);
	}
	Ok(())
}
````

```console
$ cargo run
The device "/dev/nvme0n1p3" is mounted at "/home/benjamin/Mary had".
...output trimmed for length...
```

不幸的是，在 Rust 中编写自定义迭代器需要编写大量的样板代码。与其在这里解释所有的内容，我推荐你阅读[Dan DiVica's tutorial on Rust iterators](https://dev.to/dandyvica/yarit-yet-another-rust-iterators-tutorial-46dk)。请注意：一旦我们从`BufReader`中获取到某一行，我们就再也无法回滚并重新获取到那一行的数据了。`Mounts`实现了一个消耗迭代器和可变迭代器，但是没有实现借用迭代器。来证明一下它说的是什么意思：

```rust
extern crate nom_tutorial;

fn main() -> std::result::Result<(), BoxError> {
	let mounts = nom_tutorial::mounts()?;
	
	// Do it once
	for mount in mounts {
		println!("{}", mount?);
	}
	
	// Do it again
	// Fails because we already consumed mounts in the previous for loop
	for mount in mounts {
		println!("{}", mount?);
	}
	
	// Do it again
	// Works because we get a new instance of Mounts
	// Internally works because we get a new file handle on /proc/mounts
	for mount in nom_tutorial::mounts()? {
		println!("{}", mount?);
	}
	Ok(())
}
```

```console
$ cargo check
    Checking nom-tutorial v0.1.0 (/home/benjamin/src/rust/nom-tutorial)
error[E0382]: use of moved value: `mounts`
  --> src/main.rs:12:15
   |
4  |     let mounts = nom_tutorial::mounts()?;
   |         ------ move occurs because `mounts` has type `nom_tutorial::Mounts`, which does not implement the `Copy` trait
...
7  |     for mount in mounts {
   |                  ------ value moved here
...
12 |     for mount in mounts {
   |                  ^^^^^^ value used here after move

error: aborting due to previous error

For more information about this error, try `rustc --explain E0382`.
error: Could not compile `nom-tutorial`.

To learn more, run the command again with --verbose.
```

## 结束

我希望这篇文章可以帮助到你，使你在使用 nom 时更加舒适，甚至可能学到了你之前未曾了解的 Rust 的内容。如果你发现了错别字(typo)，错误或遗漏(omission)，请不要犹豫在 Github 上打开一个 issue。Happy coding!
