# 第三方 crates

> 注意：编写过程宏并不需要自动链接的 [`proc_macro`] crate 之外的 
> crate。这里列出 crates 只是使编写它们变得更简单、更简洁，同时由于增加了依赖项，可能会增加过程宏的编译时间。

过程宏放在 crate 中，因此它们可以自然地依赖于 [crates.io](https://crates.io/) 上或其他来源的 crates。

crates 生态系统已经有一些非常实用、为过程宏量身定做的库，本章将快速介绍这些库，其中大部分将在接下来的章节中用来实现示例。

由于这些只是简单的介绍，如果真的需要使用，建议查看每个库的文档以获得更深入的信息。

> 译者注：你可以阅读我对以下这几个库的[使用总结](https://zjp-cn.github.io/rust-note/proc-macro-note.html)。

## `proc-macro2`

你可能会认为 [`proc-macro2`] 是 [`proc_macro`] 的继承者，但实际上肯定不对！

这个名字可能有点误导，因为 [`proc-macro2`] 实际上只是对 `proc_macro` 的包装，根据其文档，它用于两个特定目的：

- 将类似与过程宏的功能带到其他上下文中，如 build.rs 和 main.rs
- 让过程宏可进行单元测试

由于 `proc_macro` 只能在 `proc-macro` 类型的库中使用，所以无法直接使用 `proc_macro` 库。

始终记住，`proc-macro2` 模仿 `proc_macro` 的 api，对后者进行包装，让后者的功能在非 `proc-macro` 类型的库中也能使用。

因此，建议基于 `proc-macro2` 来开发过程宏代码的库，而不是基于 `proc_macro`
构建，因为这将使这些库可以进行单元测试，这也是以下列出的库传入和返回 [`proc-macro2::TokenStream`] 的原因。

当需要 `proc_macro::TokenStream` 时，只需对 `proc-macro2::TokenStream` 进行 `.into()` 操作即可获得 `proc_macro` 的版本，反之亦然。

使用 `proc-macro2` 的过程宏通常会以别名的形式导入，比如使用 `use proc-macro2::TokenStream as TokenStream2` 来导入 `proc-macro2::TokenStream`。

## `quote`

[`quote`] 主要公开了一个声明宏：[`quote!`](https://docs.rs/quote/*/quote/macro.quote.html)。

这个小小的宏让你轻松创建标记流，使用方法是将实际的源代码写出为 Rust 语法。

同时该宏还允许你将标记直接插入到编写的语法中：
1. 使用 `#local` 语法进行[插值]，其中 `local` 指的是当前作用域中的一个 local。[^interpolation]
2. 使用 `#(#local)*` 来对实现了 [`ToTokens`] 的类型的迭代器进行插值，其工作原理类似于声明宏的反复，因为它们允许在反复中使用分隔符和额外的标记。

[插值]: https://docs.rs/quote/*/quote/macro.quote.html#interpolation
[`ToTokens`]: https://docs.rs/quote/1/quote/trait.ToTokens.html

[^interpolation]: 译者注：这里的“插值”并不局限于插入“值或者表达式”，可以插入任何符合 Rust 语法的东西，比如标识符、条目、模块等等。

```rs
let name = /* 某个标识符 */;
let exprs = /* 某个对表达式标记流的迭代器 */;
let expanded = quote! {
    impl SomeTrait for #name { // #name 将插入上述的局部名称
        fn some_function(&self) -> usize {
            #( #exprs )+* // 通过迭代生成表达式
        }
    }
};
```

在准备输出时，`quote!` 是一个非常有用的工具，它避免了通过逐个插入标记来创建标记流。

> 注意：如前所述，此 crate 使用 `proc_macro2`，因此 `quote!` 将返回 [`proc-macro2::TokenStream`] 类型。

## `syn`

[`syn`] 是一个解析库，用于将 Rust 标记流解析为 Rust 源代码的语法树。

它是一个功能非常强大的库，使得解析过程宏输入变得非常容易，而 `proc_macro` 本身不公开任何类型的解析功能，只公开标记。

由于这个库可能是一个严重的编译依赖项，它大量使用 feature 控制来允许用户根据需要将其功能剪裁得尽可能小。

那么，它能提供什么呢？很多东西。

首先，当启用 `full` feature 时，它具有对所有标准 Rust 语法节点的定义和从而能够完全解析 Rust 语法。

在启用 `derive` feature （默认开启）之后，它还提供一个 `DeriveInput` 类型，该类型封装了传递给 derive 宏输入所有信息。

在启用 `parsing` 和 `proc-macro` feature （默认开启）之后，`DeriveInput` 可以直接与 [`parse_macro_input!`] 配合使用，以将标记流解析为所需的类型。

如果 Rust 语法不能解决你的问题，或者说你希望解析自定义的非 Rust 语法，那么这个库还提供了一个通用的[解析 API][parse]，主要是以 
[`Parse`] trait 的形式（这需要 `parsing` feature，默认启用）。

除此之外，该库公开的类型保留了位置信息和 `Span`，这让过程宏发出详细的错误消息，指向关注点的宏输入。

由于这又是一个过程宏的库，它利用了 `proc-macro2` 的类型，因此可能需要转换成 `proc_macro` 的对应类型。

> 我对 `syn` 做了更系统的梳理，你可以[阅读一下](https://zjp-cn.github.io/rust-note/proc/syn.html)。

[`DeriveInput`]: https://docs.rs/syn/*/syn/struct.DeriveInput.html
[`parse_macro_input!`]: https://docs.rs/syn/*/syn/macro.parse_macro_input.html
[parsing API]: https://docs.rs/syn/1/syn/parse/index.html
[`Parse`]: https://docs.rs/syn/1/syn/parse/trait.Parse.html

[`proc_macro`]: https://doc.rust-lang.org/proc_macro/
[`proc-macro2`]: https://docs.rs/proc-macro2
[`proc-macro2::TokenStream`]: https://docs.rs/proc-macro2/*/proc_macro2/struct.TokenStream.html 
[`quote`]: https://docs.rs/quote
[`syn`]: https://docs.rs/syn
