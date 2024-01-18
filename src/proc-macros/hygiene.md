# 卫生性和 `Span`

本章讨论过程宏的[卫生性][hygiene]以及对其进行编码的类型 [`Span`]。

[`TokenStream`] 中的每个标记都关联了一个 `Span`，它其中包含一些附加信息。

正如其文档所述，`Span` 表示“一个源代码区域，以及宏展开的信息”。

`Span` 指向原始源代码的一个区域（这对于在正确的位置显示诊断信息很重要），并保持该位置的卫生性。

卫生性主要与标识符有关，因为它允许或禁止表示符对调用外部定义的事物进行引用或者被引用。

卫生性有 3 种（这可以从 `Span` 类型的构造函数看到）：

* 定义处卫生性 ([`definition site`]) (***unstable***)： 表示宏定义处的 `Span`。带着这种 `Span`
  的标识符不能引用外部定义的内容（即这种标识符无法使用宏定义之外的内容），或者不能被外部调用的东西引用（即宏定义之外的东西无法使用这种标识符）。这就是所谓的“卫生性”。
* 混合式卫生性 ([`mixed site`])：表示宏定义处或者调用处的 `Span`，具体取决于标识符的类型。声明宏使用这种卫生性，见此[此章](../decl-macros/minutiae/hygiene.md)。
* 调用处卫生性 ([`call site`])：表示调用处的
  `Span`。此时，标识符表现得就像是直接在调用处编写的一样，也就是说，它们可以自由地使用调用之外定义的内容，也可以从外部引用它们。这就是所谓的“不卫生”
  (unhygienic)。

[hygiene]: ../syntax-extensions/hygiene.md
[`Span`]: https://doc.rust-lang.org/proc_macro/struct.Span.html
[`TokenStream`]: https://doc.rust-lang.org/proc_macro/struct.TokenStream.html
[`definition site`]: https://doc.rust-lang.org/proc_macro/struct.Span.html#method.def_site
[`mixed site`]: https://doc.rust-lang.org/proc_macro/struct.Span.html#method.mixed_site
[`call site`]: https://doc.rust-lang.org/proc_macro/struct.Span.html#method.call_site
