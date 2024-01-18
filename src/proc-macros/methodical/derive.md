# derive 式过程宏

[`derive`]: https://doc.rust-lang.org/reference/attributes/derive.html

derive 式过程宏[^derive-name]为 [`derive`] 属性定义了新的输入。这种宏通过将其名称提供给 
`derive` 属性的输入来调用，例如 `#[derive(TlbormDerve)]`。

[^derive-name]: 译者注：我通常不喜欢把 derive 翻译出来，因为它就像 trait
这个名称那样具体而明确，一目了然。当然，有时我会简写为 “derive 宏”，你称它“派生宏”也行。

一个 derive 式过程宏的简单框架如下所示：

```rs
use proc_macro::TokenStream;

#[proc_macro_derive(TlbormDerive)]
pub fn tlborm_derive(input: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

`proc_macro_derive` 稍微特殊一些，因为它需要一个额外的标识符，此标识符将成为 derive 宏的实际名称。

输入标记流是添加了 derive  属性的条目，也就是说，它将始终是 `enum`、`struct` 或者 `union` 
类型，因为这些是 derive 属性仅可以添加上去的条目。

输出的标记流将被 **追加**[^appended] 到带注释的条目所处的块或模块，所以要求标记流由一组有效条目组成。

[^appended]: 译者注：属性宏与 derive 宏的显著区别在于，属性宏生成的标记是完全替换性质，而 derive 宏生成的标记是追加性质。

用法示例：

```rs
use tlborm_proc::TlbormDerive;

#[derive(TlbormDerive)]
struct Foo;
```

# 辅助属性

derive 宏又有一点特殊，因为它可以添加仅在条目定义范围内可见的附加属性。

这些属性被称为派生宏辅助属性 (*derive macro helper attributes*) ，并且是惰性的([inert])。

[inert]: https://doc.rust-lang.org/reference/attributes.html#active-and-inert-attributes

辅助属性的目的是在每个结构体字段或枚举体成员的基础上为 derive 宏提供额外的可定制性。

也就是说这些属性可用于附着在字段或成员上，而且不会对其本身产生影响。

又因为它们是“惰性的”，所以它们不会被剥离，并且对所有宏都可见。[^active-inert]

[^active-inert]: 译者注：根据 Reference，除了属性宏的属性是 active 的，其他属性都是 inert 的。

辅助属性的定义方式是向 `proc_macro_derive` 属性增加 `attributes(helper0, helper1, ..)`
参数，该参数可包含用逗号分隔的标识符列表（即辅助属性的名称）。

因此，编写带辅助属性的 derive 宏的简单框架如下所示：

```rs
use proc_macro::TokenStream;

#[proc_macro_derive(TlbormDerive, attributes(tlborm_helper))]
pub fn tlborm_derive(item: TokenStream) -> TokenStream {
    TokenStream::new()
}
```

这就是辅助属性的全部内容。在过程宏中使用（或者说消耗）辅助属性，得检查字段和成员的属性，来判断它们是否具有相应的辅助属性。

如果条目使用了所有 derive 宏都未定义的辅助属性，那么会出现错误，因为编译器会尝试将这个辅助属性解析为普通属性（而且这个属性并不存在）。

用法示例：

```rs
use tlborm_proc::TlbormDerive;

#[derive(TlbormDerive)]
struct Foo {
    #[tlborm_helper]
    field: u32
}

#[derive(TlbormDerive)]
enum Bar {
    #[tlborm_helper]
    Variant { #[tlborm_helper] field: u32 }
}
```
