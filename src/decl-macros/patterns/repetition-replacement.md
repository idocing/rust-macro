# 反复替换

```rust,ignore
macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {$sub};
}
```

在上面代码的模式中，匹配到的重复序列将被直接丢弃，
仅留用它所带来的长度信息（以及元素的类型信息）；
且原本标记所在的位置将被替换成某种重复元素。

举个例子，考虑如何为一个元素多于12个 （Rust 1.2 下的元组元素个数的最大值）
的 `tuple` 提供默认值。

```rust,editable
macro_rules! tuple_default {
    ($($tup_tys:ty),*) => {
        (
            $(
                replace_expr!(
                    ($tup_tys)
                    Default::default()
                ),
            )*
        )
    };
}

macro_rules! replace_expr {
    ($_t:tt $sub:expr) => {
        $sub
    };
}

fn main() {
    assert_eq!(tuple_default!(i32, bool, String),
               (i32::default(), bool::default(), String::default()));
}
```

> **<abbr title="Just for this example">仅对此例</abbr>**：
我们其实可以直接用 `$tup_tys::default()` 。

上例中，我们 **并未真正使用** 匹配到的类型。
实际上，我们把它丢弃了，并用一个表达式重复替代 (repetition replacement) 。
换句话说，我们实际关心的不是有哪些类型，而是有多少个类型。
