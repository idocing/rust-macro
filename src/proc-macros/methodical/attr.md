# 属性式过程宏

属性式过程宏定义了可添加到条目的的新外部属性。这种宏通过 `#[attr]` 或
`#[attr(…)]` 方式调用，其中 `…` 是任意标记树。

一个属性式过程宏的简单框架如下所示：

```rs
use proc_macro::TokenStream;

#[proc_macro_attribute]
pub fn tlborm_attribute(input: TokenStream, annotated_item: TokenStream) -> TokenStream {
    annotated_item
}
```

这里需要注意的是，与其他两种过程宏不同，这种宏有两个输入参数，而不是一个。

* 第一个参数是属性名称后面的带分隔符的标记树，不包括它周围的分隔符。如果只有属性名称（其后不带标记树，比如 `#[attr]`），则这个参数的值为空。
* 第二个参数是添加了该过程宏属性的条目，但不包括该过程宏所定义的属性。因为这是一个 [`active`] 属性，在传递给过程宏之前，该属性将从条目中剥离出来。

[`active`]: https://doc.rust-lang.org/reference/attributes.html#active-and-inert-attributes 

返回的标记流将完全替换带被添加了该属性的条目。注意，不一定替换成单个条目，替换的结果可以是 0 或更多条目。

<!-- CONFIRM: Is this true? Can it emit an empty token stream? -->

用法示例：

```rs
use tlborm_proc::tlborm_attribute;

#[tlborm_attribute]
fn foo() {}

#[tlborm_attribute(attributes are pretty handsome)]
fn bar() {}
```
