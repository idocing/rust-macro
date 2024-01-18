# 增量式 `TT` “撕咬机”

> 译者注：原文标题为 *incremental `TT` muncher* 。

```rust
macro_rules! mixed_rules {
    () => {};
    (trace $name:ident; $($tail:tt)*) => {
        {
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
    (trace $name:ident = $init:expr; $($tail:tt)*) => {
        {
            let $name = $init;
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
}
# 
# fn main() {
#     let a = 42;
#     let b = "Ho-dee-oh-di-oh-di-oh!";
#     let c = (false, 2, 'c');
#     mixed_rules!(
#         trace a;
#         trace b;
#         trace c;
#         trace b = "They took her where they put the crazies.";
#         trace b;
#     );
# }
```

此模式可能是 **最强大** 的宏解析技巧。通过使用它，一些极其复杂的语法都能得到解析。


“标记树撕咬机” (`TT` muncher) 是一种递归宏，其工作机制有赖于对输入的顺次、逐步处理
(incrementally processing) 。处理过程的每一步中，它都将匹配并移除（“撕咬”掉）输入头部
(start) 的一列标记 (tokens)，得到一些中间结果，然后再递归地处理输入剩下的尾部。

名称中含有“标记树”，是因为输入中尚未被处理的部分总是被捕获在 `$($tail:tt)*` 
的形式中。之所以如此，是因为只有通过使用反复匹配 [`tt`] 才能做到 **无损地**
(losslessly) 捕获住提供给宏的输入部分。

标记树撕咬机仅有的限制，也是整个宏系统的局限：

* 你只能匹配 `macro_rules!` 捕获到的字面值和语法结构。
* 你无法匹配不成对的标记组 (unbalanced group) 。

然而，需要把宏递归的局限性纳入考量。`macro_rules!` 没有做任何形式的尾递归消除或优化。

在写标记树撕咬机时，建议多花些功夫，尽可能地限制递归调用的次数。

以下两种做法帮助你做到限制宏递归：
1. 对于输入的变化，增加额外的匹配规则（而不是采用中间层并使用递归）[^example]；
2. 对输入句法施加限制，以便于记录追踪标准式的反复匹配。

[^example]: 例子见 [计数-递归](../building-blocks/counting.html#递归)

[`tt`]:../minutiae/fragment-specifiers.md#tt

<a id="performance"></a>

# 性能建议

> 译者注：要点是
> 1. 可以一次处理很多标记来减少递归次数（比如运用反复匹配）
> 2. 可以编写规则简单的宏，然后多次调用
> 3. 把容易匹配到的规则放到前面，以减少匹配次数（因为规则顺序决定了匹配顺序）

TT 撕咬机天生就是二次复杂度的。考虑一个 TT 撕咬机
规则，它消耗一个标记树，然后递归地在其余输入上调用自身。如果向其传递 100 个标记树：

- 初始调用将匹配所有的 100 个标记树。
- 第 1 个递归调用将匹配 99 个标记树。
- 下一次递归调用将匹配 98 个标记树。
- 依此类推，直到匹配最后 1 个标记树。

这是一个典型的二次复杂度模式，过长的输入会导致宏展开延长编译时间。

因此，尽量避免过多地使用 TT 撕咬机，特别是在输入较长的情况下。

`recursion_limit` 属性的缺省值 (目前是 128 )
是一个良好的健全性检查；如果你必须超过它，那么可能会遇到麻烦。

建议是，你可以选择编写一个：
1. 一次调用就能处理多件事情的 TT 撕咬机
2. 或者多次调用来处理一件事情的更简单的宏（这种宏从性能角度看，是更推荐的做法）

例如，别这样写：

```rust
# macro_rules! f { ($($tt:tt)*) => {} }
f! {
    fn f_u8(x: u32) -> u8;
    fn f_u16(x: u32) -> u16;
    fn f_u32(x: u32) -> u32;
    fn f_u64(x: u64) -> u64;
    fn f_u128(x: u128) -> u128;
}
```

应该这样写：

```rust
# macro_rules! f { ($($tt:tt)*) => {} }
f! { fn f_u8(x: u32) -> u8; }
f! { fn f_u16(x: u32) -> u16; }
f! { fn f_u32(x: u32) -> u32; }
f! { fn f_u64(x: u64) -> u64; }
f! { fn f_u128(x: u128) -> u128; }
```

宏的输入越长，第二种编写方式就越有可能缩短编译时间。

此外，如果 TT 撕咬机有许多规则，请 **尽可能把最频繁匹配的规则放到前面**
。这避免了不必要的匹配失败。（事实上，这对任何类型的声明性宏都是很好的建议，而不仅仅是
TT 撕咬机。）

最后，优先使用正常的反复匹配（`*` 或 `+`）来编写宏，这比 TT 撕咬机更好。如果每次调用
TT 撕咬机时，一次只处理一个标记，则最有可能出现这种情况。

在更复杂的情况下，可以参考 `quote!` 使用的一种高级技术，它可以避免二次复杂度，而且不会达到递归上限，但代价是一些复杂的概念。详情请参考[此处][quote]。

[quote]: https://github.com/dtolnay/quote/blob/31c3be473d0457e29c4f47ab9cff73498ac804a7/src/lib.rs#L664-L746

