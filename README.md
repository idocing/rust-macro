# Rust 宏小册

> 注意：这是对 [Daniel Keep 撰写的书](https://github.com/DanielKeep/tlborm)
> 的续写，自 2016 年初夏以来，那本书就没再更新。

本书的续写者为 [Veykril](https://github.com/veykril)，使用 
[mdBook](https://github.com/rust-lang/mdBook) 工具生成。你可以浏览本书的
[英文版本](https://veykril.github.io/tlborm/)，和 github
[仓库](https://github.com/veykril/tlborm)。[^translation-statement]

这本书尝试提炼出 Rust 社区对 Rust 宏的共识，准确地说，是 *通过例子* 来讲述宏[^macros]。
因此，欢迎 PR 补充和提 issue。

如果你对某些书中的内容不清楚，或者不理解，别害怕提 issue
来请求澄清那部分。本书的目标是尽可能成为最好的（宏）学习资料。

在我学习 Rust 的时候，*Little Book of Rust Macros* [原作](https://github.com/DanielKeep/tlborm) 
*通过例子* 的方式非常给力地帮助过我理解（声明）宏。很遗憾的是，Rust
语言与宏系统持续改进时，原作者不再更新书籍。

这也是我想尽可能地更新这本书的原因，并且我尽可能地把新发现的事情增加到书中，以帮助新的
Rust 宏学习者理解宏系统 —— 这个让很多人困惑的部分。

> 这本书认为你应该对 Rust 有基本的了解，它不会解释 Rust 
> 语言特性或者与宏无关的结构，但不会假设你提前掌握宏的知识。
>
> 你必须至少阅读和理解了 [Rust Book](https://doc.rust-lang.org/stable/book/) 
> 的前七章 —— 当然，建议你阅读完 Rust Book 的大部分内容。

[^translation-statement]:译者注：我对原作和续作进行了梳理，见 [翻译说明](./translation_statement.html)。

[^macros]: 译者注：2022 年的中文版随续作更新了过程宏，而声明宏也一直在演进。

## 致谢

非常感谢 [Daniel Keep](https://github.com/DanielKeep/tlborm) 最初写下这本书。[^thanks]

感谢对原书提出建议和更正的读者：
IcyFoxy、 Rym、 TheMicroWorm、 Yurume、 akavel、 cmr、 eddyb、 ogham 和 snake_case。

[^thanks]: 译者注：非常感谢 Veykril 不懈地更新此书。感谢
[DaseinPhaos](https://github.com/DaseinPhaos/tlborm-chinese) 对原作的翻译。此外，本书的右侧
TOC 是由 [mdbook-theme](https://github.com/zjp-CN/mdbook-theme) 所提供。

## 版权声明

这本书沿袭了原作的版权声明，因此具有 [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/) 和 [MIT license](http://opensource.org/licenses/MIT) 的双重许可。
