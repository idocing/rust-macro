# 思路介绍

本章将从整体角度解释和介绍过程宏。

与[声明宏]不同，过程宏采用 Rust 函数的形式，接受一个（或两个）标记流并输出一个标记流。

[声明宏]: ../decl-macros.md

过程宏的核心只是一个从 `proc-macro` [crate type](https://doc.rust-lang.org/reference/linkage.html) 这种类型的库中所导出的公有函数，因此当编写多个过程宏时，你可以将它们全部放在一个 crate 中。

> 注意：在使用 Cargo 时，定义一个 `proc-macro` crate 的方式是将
> `Cargo.toml` 中的 `lib.proc-macro` 键设置为 true，就像这样
> 
> ```toml
> [lib]
> proc-macro = true
> ```

`proc-macro` 类型的 crate 会隐式链接到编译器提供的 [proc_macro](https://doc.rust-lang.org/proc_macro/index.html) 库，
proc_macro 库包含了开发过程宏所需的所有内容，并且它公开了两个最重要的类型：
1. [`TokenStream`](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html)：它表示我们所熟知的标记树
2. [`Span`](https://doc.rust-lang.org/proc_macro/struct.Span.html)：它表示源代码的一部分，主要用于错误信息的报告和卫生性，更多信息请阅读
   [卫生性和 Spans](./hygiene.md) 一章

因为过程宏是存在于 crate 中的函数，所以它们可以像 Rust 项目中的所有其他条目一样使用。

使用过程宏只需要将 proc-macro 类型的 crate 添加到项目的依赖关系图中，并将所需的过程宏引入作用域。

> 注意：调用过程宏与编译器展开成声明宏是在同一阶段运行，只是过程宏是编译器编译、运行、最后替换或追加的独立的 Rust 程序。

## 过程宏的类型

过程宏实际上存在三种不同的类型，每种类型的性质都略有不同。[^kinds]

* 函数式：实现 `$name！$input` 功能的宏
* 属性式：实现 `#[$input]` 功能的属性
* derive 式：实现 `#[derive($name)]` 功能的属性

[^kinds]: 译者注：你可以参考我[总结的表](https://zjp-cn.github.io/rust-note/proc/FAQ.html#%E4%BB%80%E4%B9%88%E6%98%AF%E8%BF%87%E7%A8%8B%E5%AE%8F)

### 函数式

```rust,ignore
#[proc_macro]
pub fn name(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

### 属性式

```rust,ignore
#[proc_macro_attribute]
pub fn name(attr: TokenStream, input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

### derive 式

```rust,ignore
#[proc_macro_derive(Name)]
pub fn my_derive(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

如上所示，每个函数的基本结构是相同的：一个标记了一个属性的公有函数，这个属性定义了它的过程性宏类型，然后函数返回一个 `TokenStream`。

注意，返回类型必须是一个 `TokenStream`[^TokenStream]。

过程宏也会失败，它们有两种报告错误的方式：
1. panic：此时编译器会捕获到，然后把它作为来自于宏调用的错误发出
2. 调用 [`compile_error!`](https://doc.rust-lang.org/std/macro.compile_error.html) 

> 注意：如果过程宏内出现无限循环，编译器会长时间等待（挂起），从而造成使用过程宏的 crate 也编译挂起。

[^TokenStream]: 译者注：而且这个 `TokenStream` 类型必须是 `proc_macro` 所公开的 `TokenStream`，通常使用 `quote` 库构造这种类型。
