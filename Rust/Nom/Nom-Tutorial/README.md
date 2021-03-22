# Nom5教程

[原文](https://github.com/benkay86/nom-tutorial)

[Nom](https://github.com/Geal/nom)是一个用 Rust 编写的解析器组合器库。它可以处理二进制和文本文件。在需要使用正则表达式，Flex或Bison的地方可以考虑使用它。Nom具有 Rust 的强类型和内存安全的优势，它通常比其他可替代的工具性能更好。学习 Nom 是 Rust 工具箱中的一项有价值的补充。

## 基本原理

Nom 的官方文档包含一些简单的示例(如：如何解析十六进制RGB颜色代码)以及非常复杂的示例(如：如何解析json)。当我第一次学习 Nom 时，我在简单示例和复杂示例之间发现了陡峭的学习曲线。此外，Nom 的早期版本和大多数现有文档都使用宏。从 Nom 5.0 开始，不赞成使用宏，而推荐使用函数。本教程旨在通过解析`/proc/mounts`的内容来填补简单解析器和复杂解析器之间的空白，并演示使用函数替代宏的方法。

## Table of Contents

1. [The Exercise](#chap1)
2. [Getting Started](#chap2)
3. [Hello Parser](#chap3)
4. [Reading the Nom Documentation](#chap4)
5. [Laying the Groundwork](#chap5)
6. [It's Not Whitespace](#chap6)
7. [The Great Escape](#chap7)
8. [Mount Options](#chap8)
9. [Putting it All Together](#chap9)
10. [Iterators are the Finishing Touch](#chap10)

## <a name="chap1"></a>The Exercise

如果你使用的是 Linux 系统，你可能对`mount`命令很熟悉了。如果你运行`mount`却不带任何参数，它将会在终端上打印出已挂载文件系统的清单。

```console
$ mount
sysfs on /sys type sysfs (rw,seclabel,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
...output trimmed for length...
```

使用 Rust 实现`mount`命令的全部功能超出了本教程的范围，但是我们可以借助 Nom 实现上面的输出内容。Linux 内核在`proc/mounts`中存储了当前所有关于已挂载文件系统的信息。

```console
$ cat /proc/mounts
sysfs /sys sysfs rw,seclabel,nosuid,nodev,noexec,relatime 0 0
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0
```

每个 mount 都在单独的一行中描述。在每一行中，mount 的属性使用空格分开。

1. 设备 (如：sysfs, /dev/sda1)
2. 挂载点 (如：/sys, /mnt/disk)
3. 文件系统类型 (如：sysfs, ext4)
4. 挂载选项，一个用逗号分隔的字符串 (如：rw, ro)
5. 每一行以`0 0`结尾，以模仿(mimic)`/etc/fstab`的格式。第五列的`0 0`只是一个装饰 -- 每行都相同，因此不包含有用的信息。

__在本教程中__，我们会写一个程序将`/proc/mounts`的每一行解析成 Rust 的结构体并将它们打印在控制台上，就像`mount`命令一样。


## <a name="chap2"></a>Getting Started

To learn from the example code you will need to [have Rust installed](https://www.rust-lang.org/learn/get-started), and I will assume you have some basic familiarity with the Rust language.  To download and run the complete tutorial:

```console
$ git clone https://github.com/benkay86/nom-tutorial.git
$ cd nom-tutorial
$ cargo run
sysfs on /sys type sysfs (rw,seclabel,nosuid,nodev,noexec,relatime)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
...output trimmed for length...
```
The finished version of the tutorial is a lot to digest at once, so in the sections below we will build up to it step-by-step.  I recommend creating your own cargo project to experiment with `cargo new my-nom-tutorial` and keeping a copy of the completed tutorial as a reference.  To use nom in your own cargo package simply edit `Cargo.toml` to contain:

```toml
[dependencies]
nom = "5.0"
```

## <a name="chap3"></a>Hello Parser

In your new project, edit `main.rs` to contain the following:

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
Compile and run the program:

```console
$ cargo run
Ok(("", "hello"))
Ok((" world", "hello"))
Err(Error(("goodbye hello again", Tag)))
```
Let's break this program down line by line.

### Using the Nom Crate

```rust
extern crate nom;
```
In the [previous section](#chap2) we added nom as a dependency in `Cargo.toml`.  This additional line in `main.rs` tells your program about the nom crate, enabling you to access it through `nom::`.  You can optionally add lines like `use nom::IResult;` to cut down on typing, but I have deliberately used the verbose notation in this tutorial so that you can clearly see the module hierarchy.

### Creating a Custom Parser

```rust
fn hello_parser(i: &str) -> nom::IResult<&str, &str> {
	nom::bytes::complete::tag("hello")(i)
}
```
This creates a function called `hello_parser` that takes a `&str` (borrowed string slice) as its input and returns a type `nom::IResult<&str, &str>`, which we'll talk more about later.  Within the body of the function we create a nom tag parser.  A tag parser recognizes a literal string, or "tag", of text.  The tag parser `tag("hello")` is a function object that recognizes the text "hello".  We then call the tag parser with the input string as its argument and return the result.  (Remember, in Rust you can omit the `return` keyword from the last line in a function.)

### Invoking the Parser

```rust
println!("{:?}", hello_parser("hello world"));
// Ok((" world", "hello"))
```
Now let's go to `main()` and see what the parser does.  Recall that `println!("{:?}", x)` prints out the debugging version of `x`, giving us an easy way to inspect the content of Rust variables.  Here we call `hello_parser()` with several different test strings and print out the returned `nom::IResult<&str, &str>`.  As you can see, it turns out an `IResult` is a Rust `Result`, which can contain an `Ok` or `Err`.  When the parser succeeds it returns a tuple of its generic type parameters, in this case `&str`
.  The second element of the tuple is the "output" of the parser, which is often the string matched or "consumed by" the parser, "hello".  The first element of the tuple is the remaining input, " world".

```rust
println!("{:?}", hello_parser("hello"));
// Ok(("", "hello"))
```
In this case the tag consumes the whole input, so the first element of the tuple (the remaining input) is an empty string.

```rust
println!("{:?}", hello_parser("goodbye hello again"));
// Err(Error(("goodbye hello again", Tag)))
```
Here the tag returns an `Err` because the input string didn't start with "hello."  Note that the parser failed even though the word "hello" appears in the middle of the input -- most nom parsers (including tag) will only match the beginning of the input.  The `Error` object is a `nom::Err::Error((&str, nom::error::ErrorKind))`, which is a tuple of the remaining input (the parser failed, so all of the input remained) and an `ErrorKind` describing which parser failed.  You can read more about [advanced nom error handling on github](https://github.com/Geal/nom/blob/master/doc/error_management.md).

### Summary

* Nom parsers typically take an input `&str` and return an `IResult<&str,&str>`.
* You can compose your own parser by defining a `fn (&str) -> IResult<&str,&str>` that returns the result of some combination of nom parsers.
* When a parser successfully matches some or all of the input it returns `Ok` with a tuple of the remaining input and the consumed input.
* When a parser fails to match any input it returns an `Err`.
* Most nom parsers match only the beginning of the input, even if there is a pattern that could match later in the input.

## <a name="chap4"></a>Reading the Nom Documentation

You will need to refer to the [documentation for nom](https://docs.rs/nom) often.  Make sure you are reading the documentation for version 5.0 or later, since a lot has changed since version 4.  Previous versions of nom were very macro centric, so you will find a lot of references to macros like `tag!()`.  Macros have been soft-deprecated in favor of functions.  Most functions have the same name as their macro counterparts but without the exclamation point, i.e. `tag()`.  You can see a [list of all nom's functions here](https://docs.rs/nom/5.0.0/nom/all.html#Functions).

You will find that there are `streaming` and `complete` submodules.  In advanced use, nom supports streaming, or buffered, input where the parser might encounter incomplete fragments of input.  In this tutorial we will focus on the `complete` submodule for non-streaming input.

* `nom::branch` parsers perform logical operations on multiple sub-parsers.  For example, `nom::branch::alt` succeeds if any one of its sub-parsers succeeds.
* `nom::bytes::complete` parsers operate on sequences of bytes.  Our friend `tag` belongs to this submodule.
* `nom::character::complete` recognizes characters, for example `nom::character::complete::multispace1` matches 1 or more characters of whitespace.
* `nom::combinator` allows us to build up combinations of parsers.  For example, `nom::combinator::map` passes the output of one parser into a second parser.
* `nom::multi` parsers return collections of outputs.  For example, `nom::multi::separated_list` returns a vector of strings separated by a delimiter.
* `nom::number::complete` parsers match numeric values.
* `nom::sequence` parsers match finite sequences of input.  For example, `nom::sequence::tuple` takes a tuple of sub-parsers and returns a tuple of their outputs.

## <a name="chap5"></a> Laying the Groundwork

This section deals with setting up the non-nom (isn't that fun to read out load?) parts of the program.  If you are already quite familiar with rust and just want to read about nom then [skip to the next section](#chap6).

### Encapsulation

It is simple, and tempting, to write your whole program in one file.  However, it is good practice to split your program into a library (or crate) and binary to make the underlying logic easy to reuse.  We'll take the high road in this tutorial and create an empty file called `lib.rs` in the same directory as `main.rs`.  Cargo automatically knows to build `lib.rs` into a library/crate with the name "nom-example" we specified in `Cargo.toml` using the line `name = "nom-example"`.  Then let's make a new `main.rs` that uses our `nom-example` crate instead of using nom directly.

```rust.rs
extern crate nom_example;

fn main() {
}
```
Note that when the name of a crate contains hyphens we replace them with underscores in the Rust code.

### Error Handling

Unfortunately, many Rust tutorials handle potential errors by having you write `could_fail.unwrap()` or `could_fail.expect("Oh no!")`.  These statements cause your code to panic whenever an error occurs.  That's all well and good in a simple didactic example, but you should avoid writing production code that panics.  Instead we will introduce the syntax `could_fail?` known as the question mark `?` operator.  This requires a bit of plumbing.

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


Our `main()` function now returns a `Result` in which the error type is something called `BoxError` that we defined in `lib.rs`.  At the end of `main()` we return `Ok(())` with an empty tuple to signify successful completion of the program.  When we write `main()` or any other function in this way it allows us to write `could_fail?` which behaves similarly to `could_fail.unwrap()` except that it returns an error up the call stack instead of panicking.  Refer to the [Rust Book section on error handling](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html) if you are not familiar with this syntax.

What exactly is this mysterious `BoxError`?  It's something called a [trait object](https://doc.rust-lang.org/reference/types/trait-object.html) which, in this case, allows you to pass any error implementing the standard [`Error`](https://doc.rust-lang.org/std/error/trait.Error.html) trait up the call stack.  Note the inclusion of [`Send`](https://doc.rust-lang.org/std/marker/trait.Send.html) and [`Sync`](https://doc.rust-lang.org/std/marker/trait.Sync.html) to require that errors be thread-safe; although the usefulness of this may not be apparent now, it becomes very important whenever you interface with concurrent code or libraries.  Refer to [my error tutorial](https://benkay86.github.io/rust-error-tutorial.html) to learn more about this design pattern.

> Note: In previous versions of this tutorial I demonstrated how to write a custom error type for encapsulating nom errors.  [Since 5.1.1 nom errors implement `Error`](https://github.com/Geal/nom/pull/1043) and thus work out-of-the-box with the Rust's question mark `?` operator.  Writing custom error types is no longer needed.  Hooray!

### Storing the Mount Information

When we parse a line in `/proc/mounts` we are going to want to parse it _into_ something.  Let's add a simple struct to `lib.rs` for storing the information about a mount.  Note that we could use a [HashSet](https://doc.rust-lang.org/std/collections/struct.HashSet.html) for the mount options but will instead use a vector for simplicity.

```rust
#[derive(Clone, Default, Debug)]
pub struct Mount {
	pub device: std::string::String,
	pub mount_point: std::string::String,
	pub file_system_type: std::string::String,
	pub options: std::vec::Vec<std::string::String>,
}
```

## <a name="chap6"></a>It's Not Whitespace

Building a parser with nom is a lot like building with legos.  You start with building the smallest piece and then gradually combine pieces together until you get a cool looking castle or spaceship.  You'll recall that each line in `/proc/mounts` is whitespace-delimited:

```console
sysfs /sys sysfs rw,seclabel,nosuid,nodev,noexec,relatime 0 0
```
That means that each item within the line is simply a sequence of characters/bytes that is _not_ whitespace.  We'll start by making a nom parser that recognizes any sequence of one or more bytes that is not whitespace.

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
The core of this parser is `nom::bytes::complete::is_not(" \t")` which is a nom parser that recognizes one or more bytes that is not a space or tab -- i.e. is not whitespace, exactly what we want!  If the syntax for creating a custom parser (here named `not_whitespace`) doesn't look familiar to you then go back to the [Hello Parser](#chap3) example.

### Organization

Although not strictly necessary to make a program work, we try to model good coding practices through encapsulation.  We'll put all our nom parsers inside a submodule named `parsers`.  The submodule is `pub(self)`, which means that other methods in `lib.rs` can use it but it's not exposed outside of our crate.

One of the parsers we write later on will need to use the `Mount` struct we defined in the [previous section](#chap5).  We use `use super::Mount` to make the `Mount` struct defined in the parent, or "super" scope of the `parsers` module visible inside the `parsers` module.

### Unit Tests

We also model another good programming practice, unit testing.  Within the `parsers` module we've defined another submodule called `tests` (you could call it anything you want).  The line `#cfg[(test)]` tells Cargo that the `tests` module should only be compiled when running `cargo test`.  The actual test takes place inside the function `fn test_not_whitespace()` (which again can have any name, but let's not get too creative).  The `#[test]` just before the function name tells Cargo to run that function as a unit test when invoked with `cargo test`.

Here panics are OK.  A unit test succeeds if it doesn't panic.  The macro `assert_eq!()` panics if its two arguments aren't equal.  We test out a few assertions in which the `not_whitespace` parser should succeed and make sure that the whitespace and following characters in each input sequence are not consumed.  We also test out one case where the parser should fail.  Even though our program isn't finished yet, you can already compile it and make sure the `not_whitespace` parser works as expected:

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

## <a name="chap7"></a>The Great Escape

What happens if we mount a directory with spaces?  If you have root access you can try the following, otherwise take my word for it.

```console
$ mkdir "Marry had"
$ mkdir "a little lamb"
$ sudo mount -o bind "a little lamb" "Mary had"
$ cat /proc/mounts
/dev/nvme0n1p3 /home/benjamin/Mary\040had btrfs rw,seclabel,noatime,nodiratime,ssd,discard,space_cache,subvolid=258,subvol=/home/benjamin/a\040little\040lamb 0 0
...output trimmed for length...
```
As you can see, each space was replaced with `\040`.  This is a feature common to many languages you might have to parse called an escaping.  The character `\` is the escape character and `040` is the escaped sequence.  Sometimes you might actually want a `\` to appear in which case you would escape it as `\\`.

Fortunately, nom already has a built-in parser for dealing with escaped sequences called `nom::bytes::complete::escaped_transform`.  As the name implies, it transforms each escaped sequence of bytes into a literal sequence of bytes.

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

### Start Simple

We start by defining custom parsers `escaped_space` and `escaped_backslash` that recognize their escaped sequences, `040` and `\`, and return the un-escaped sequences ` ` and `\`, respectively.

The `escaped_space` parser uses `nom::combinator::value`, which returns the specified value (in this case a space) when its child parser (in this case the familiar `tag`) succeeds.  We could have written it this way:

```rust
fn escaped_space(i: &str) -> nom::IResult<&str, &str> {
	match nom::bytes::complete::tag("040")(i) {
		Ok((remaining_input, _)) => Ok((remaining_input, " ")),
		Err(e) => Err(e)
	}
}
```
But nom provides us with a lot of convenient parsers like `combinator::value` out-of-the-box to make our lives easier.

### Combining Parsers

With our simpler sub-parsers written and tested, it is now easy to use the `escaped_transform` parser.  If we were only escaping `\040` and didn't care about `\\` then we could have written it as:

```rust
nom::bytes::complete::escaped_transform(nom::bytes::complete::is_not("\\"), '\\', escaped_space)(i)	
```
`escaped_transform` takes two parsers and a `char` as arguments:

1. A sequence of bytes that is not escaped.  In our case we can use the familiar `bytes::complete::is_not` parser to match one or more bytes that is not the escape character.
2. The escape character itself, `\`.
3. A parser that transforms the escaped sequence (minus the preceding `\`) into its final form.

In our example we have multiple escaped sequences to deal with, so we use `nom::branch::alt`, which takes a tuple of parsers as  arguments and returns the result of whichever one matches first:

```rust
escaped_transform(..., alt((escaped_backslash, escaped_space)))
```

### Return Types

Up until now we've seen nom parsers return an `IResult<&str, &str>`, but nom parsers are just Rust functions and they can return anything.  If you've studies the example code closely you've noticed:

```rust
fn transform_escaped(i: &str) -> nom::IResult<&str, std::string::String>
```
This is because the `escaped_transform` parser can't generate its output string without copying/allocating memory, so instead of an `&str` it returns `nom::IResult<&str, std::string::String>`.

## <a name="chap8"></a>Mount Options

We're almost there.  We have to define one more custom parser before we assemble all our custom parsers into (metaphorically) a glorious lego spaceship.  The following custom parser transforms a comma-separated list of mount options like `ro,user` into a vector of strings like `["ro", "user"]`.  By now it should be fairly obvious to you what this code does and how it works:

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
As you can see from the return type of `mount_opts` we are going to generate a `Vec<String>` just like we promised.  The parser `multi::separated_list` does just that, parsing a list separated by some parser with elements that match some other parser into a vector.

1. The list is separated by `character::complete::char(',')`.
2. The elements of the list must not contain commas.  They also must not contain whitespace because the list is terminated by whitespace.
3. While we're at it, we use `combinator::map_parser` to call the `transform_escaped` parser on the output of `is_not(", \t")` before adding it to the vector.  This allows us to conveniently deal with the escaped characters in one fell swoop.

## <a name="chap9"></a>Putting it All Together

This tutorial may have felt like a lot of coding with no end in sight.  Now that we've defined all the custom parsers we need, we will write one more parser that puts everything together.  Hopefully when you see how simple it is to compose high-level parsers from simple parsers you will appreciate how powerful your programs will be when you use nom.

### The Final Parser

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
Wow, that's a lot of code!  Taking a birds-eye view, notice that `parse_line` returns a `Mount`.  Also notice that it's `pub` since this is the one parser we'll want to call from outside the `parsers` module.  Let's break up the details into 3 parts (labeled by comments in the code):

1. Ignore the `all_consuming` parser for now, `sequence::tuple` matches a tuple of sub-parsers in order.  In part 1 we supply a list of child parsers (as a tuple) that we want to match.  This allows us to tell nom what a line in `/proc/mounts` should look like: first some non-whitespace, then some whitespace, then some more non-whitespace, then some more whitespace, at some point some mount options, and so forth.  Note how we slipped in some calls to `map_parser` with `transform_escaped` to deal with escaped characters.

2. The `sequence::tuple` parser returns a tuple where each element in the tuple corresponds to each of its child parsers.  In part 2 we destructure the tuple into some descriptively names local variables.  For example, the very first non-whitespace sequence on a line is the device, so we destructure the first element in the tuple to a variable called `device`.  We ignore elements of the tuple we don't care about (like the whitespace) by using `_` as a placeholder.
3. We create and then return a new `Mount` object using the local variables desctructured in part 2.

Finally, the `all_consuming` parser fails if there is any input left over.  This will cause `parse_line` to (conservatively) return an error if there is something at the end of the line we were not expecting.

### Alternative Final Parser

I've received what I think is valid feedback that the final parser above is too complicated to look at.  What follows is an alternative version of the final parser that accomplishes the same objective with fewer, possibly more readable (depending on your sensibilities) lines of code.  It makes heavy use of the `?` operator to break the `tuple` parser into individual statements.  The `?` operator ends the function early, returning an error, if a parser fails.  The remaining input from each parser is used as the input of the next parser.  Pertinent variables are stored and later used to construct the `Mount` object at the end of the function.  Superfluous variables are discarded by assigning to `_`.

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

Try it out for yourself by commenting-out the original function and renaming `parse_line_alternate` to `parse_line`.  Use whichever style you like better in your own code.

### Testing It Out

You can already verify the program works with `cargo test` but let's make things a little nicer so that calling our binary will display a line-by-line list of mounts.  We'll define a function `nom_tutorial::mounts()` to print them out and then call it from `main.rs`.

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
We open the file `/proc/mounts`, created a `BufReader` to read it line-by-line, and then parse each line.  If parsing leads to an error we convert that into our custom error type `ParseError` [defined earlier](#chap5).  If parsing is successful (which it should be) we print the `Mount` option out on a new line.  To try it out:

```console
$ cargo run
/dev/nvme0n1p3 on /home/benjamin/Mary had type btrfs (rw,seclabel,noatime,nodiratime,ssd,discard,space_cache,subvolid=258,subvol=/home/benjamin/a little lamb)
...output trimmed for length...
```
We could have read the entire contents of `/proc/mounts` and used `nom::character::complete::line_ending` to modify our parsers to recognize the line endings.  However, what if `/proc/mounts` was very long?  Maybe we are working on a big server with hundreds of mounted filesystems leading `/proc/mounts` to be hundreds of megabytes in size!  (OK, that probably wouldn't happen in real life.)  Since Rust already gives us another way to parse line endings (the `BufReader`) we might as well take advantage of it to lower our (theoretical) memory use and keep our parser simple.


## <a name="chap10"></a>Iterators are the Finishing Touch

From the standpoint of splitting our parser into a library and a binary, simply having a function `mounts()` that prints out a list of mounts isn't very ergonomic.  The final version of this tutorial, which you can download from Github, introduces a new object of type `Mounts` that internally manages a `BufReader` on `/proc/mounts` and implements the `IntoIterator` trait.  This enables us to write `main.rs` like this:

```rust
extern crate nom_tutorial;

fn main() -> std::result::Result<(), BoxError> {
	for mount in nom_tutorial::mounts()? {
		println!("{}", mount?);
	}
	Ok(())
}
```
To see how powerful this is we can play around a little:

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
Unfortunately, there is a fair bit of boilerplate code needed to write a custom iterator in Rust.  Rather than try to explain it all here I recommend you read [Dan DiVica's tutorial on Rust iterators](https://dev.to/dandyvica/yarit-yet-another-rust-iterators-tutorial-46dk).  Note that once we get a line from `BufReader` we can't rewind and get the line again.  Therefore, `Mounts` implements a consuming iterator and a mutable iterator, but it doesn't implement a borrowed iterator.  To demonstrate what that means:

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

## Closing

I hope this tutorial has helped you feel comfortable using nom, and maybe even learned a little bit more about Rust than you knew before.  Please don't hesitate to open an issue on Github if you discover typos, errors, or omissions.  Happy coding!
