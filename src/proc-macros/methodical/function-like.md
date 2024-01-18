# 函数式过程宏

类似函数的过程宏，像声明宏那样被调用，即 `makro!(…)`。

不过，这类宏是三种宏中最简单的一种。它也是唯一一个在单独看调用形式时，无法与声明宏区分开的宏。

类似函数式过程宏的简单编写框架如下所示：

```rs
use proc_macro::TokenStream;

#[proc_macro]
pub fn tlborm_fn_macro(input: TokenStream) -> TokenStream {
    input
}
```

[`TokenStream`]: https://doc.rust-lang.org/proc_macro/struct.TokenStream.html

可以看到，这实际上只是从一个 [`TokenStream`] 到另一个 `TokenStream` 的映射，其中
`input` 是调用分隔符内的标记。

例如，对于示例调用 `foo!(bar)`，输入标记流将由单独的 `bar` 标记组成。返回的标记流将替换宏调用。

这种宏类型与声明宏具有相同的放置和展开规则，即宏必须在调用位置上输出正确的标记流。

但是，与声明性宏不同，函式过程宏对其输入没有特定的限制。也就是说，在 [再谈元变量与宏展开](../../decl-macros/minutiae/metavar-and-expansion.md)
一章中列出的片段分类符跟随限制在这里不适用，因为过程宏直接作用于标记，而不是根据片段分类符或类似的东西（比如反复）匹配它们。

话虽如此，很明显，过程宏更强大，因为它们可以任意修改其输入，并生成任何所需的输出，只要输出在 Rust 的语法范围内。

用法示例：

```rs
use tlborm_proc::tlborm_attribute;

fn foo() {
    tlborm_attribute!(be quick; time is mana);
}
```
