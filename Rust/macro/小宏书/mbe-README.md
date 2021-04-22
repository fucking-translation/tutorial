% 宏，彻底剖析

本章节将介绍Rust的“示例宏”(Macro-By-Example)系统:`macro_rules`。我们不会通过实际的示例来介绍它，而将尝试对此系统的**运作方式**给出完备且彻底的解释。因此，本章的目标读者应是那些想要理清这整个系统的人，而非那些仅仅想要了解它一般该怎么用的人。

在[Rust官方教程中也有一章讲解宏](http://doc.rust-lang.org/book/macros.html)([中文版](https://kaisery.gitbooks.io/rust-book-chinese/content/content/Macros%20%E5%AE%8F.html))，它更易理解，提供的解释更加高层。本书也有一章[实践介绍](pim-README.md)，旨在阐释如何在实践中实现一个宏。
