# 声明宏

本章将介绍 Rust 的声明宏系统：[`macro_rules!`](https://doc.rust-lang.org/reference/macros-by-example.html)。

在这一章中有两种不同的介绍，一个 [讲思路](./decl-macros/macros-methodical.md)，另一个
[讲实践](./decl-macros/macros-practical.md)。

前者会向你阐述一个完整而详尽的系统如何工作，而后者将涵盖更多的实际例子。

因此，[思路介绍](./decl-macros/macros-methodical.md)
是为那些只希望声明宏系统作为一个整体得到解释的人而设计的，而
[实践介绍](./decl-macros/macros-practical.md) 则指导人们通过实现单个宏。

在这两个介绍之后，本章还提供了一些常规且有用的 [模式](./decl-macros/patterns.md)
和 [构件](./decl-macros/building-blocks.md)，用于创建功能丰富的声明宏。

关于声明宏的其他资源：
1. Rust Book 的[宏章节](https://doc.rust-lang.org/book/ch19-06-macros.html)，这是一个更平易近人的高级解释
2. Reference [macros-by-example](https://doc.rust-lang.org/reference/macros-by-example.html)
   章节，它更深入而精确地讨论了细节

 > 注意：本书在讨论声明宏时，通常会使用术语 *mbe* (**M**acro-**B**y-**E**xample)、 *mbe macro* 或
 > `macro_rules!`。
