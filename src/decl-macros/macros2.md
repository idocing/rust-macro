# 声明宏 2.0

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/1584-macros.md)  
> *Tracking Issue*: [rust#39412](https://github.com/rust-lang/rust/issues/39412)  
> *Feature*: `#![feature(decl_macro)]`

虽然这还未稳定（或者更确切地说，还远未完成），但有人提议建立一个新的声明宏系统，该系统应该取代
`macro_rules!`，并给其取名为声明宏 2.0、`macro`、`decl_macro` 或者更混乱的名称 `macros-by-example`。 

本章只是为了快速浏览当前状态，展示如何使用这个宏系统以及它的不同之处。

这里所描述的一切都不是最终成型的或完整的，因为它们可能会发生变化。

## 语法

我们将对前几章中实现的两个宏在 `macro` 和 `macro_rules` 的语法之间进行比较：

```rust,editable
#![feature(decl_macro)]

macro_rules! replace_expr_ {
    ($_t:tt $sub:expr) => { $sub }
}
macro replace_expr($_t:tt $sub:expr) {
    $sub
}

macro_rules! count_tts_ {
    () => { 0 };
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 };
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 };
}
macro count_tts {
    () => { 0 },
    ($odd:tt $($a:tt $b:tt)*) => { (count_tts!($($a)*) << 1) | 1 },
    ($($a:tt $even:tt)*) => { count_tts!($($a)*) << 1 },
}

fn main() {}
```

它们看起来非常相似，只是有一些不同之处，而且 `macro` 有两种不同的形式。

让我们先看 `count_tts` 宏，因为它看起来更像我们习惯看到的样子。虽然它看起来与
`macro_rules` 的版本几乎相同，但有两个不同之处：
1. 它使用了 `macro` 关键字
2. 规则分隔符是 `,` 而不是 `;`

不过，`macro` 还有另一种形式，这是只有一条规则的宏的简写。通过
`replace_expr`，我们看到，可以用一种更类似于普通函数的方式来编写定义：
1. 直接在宏名字后面编写 matcher
2. 然后去掉一对大括号和 `=>`，再写 transcriber

调用 `macro` 所定义的宏，和函数式宏的语法相同，名称后跟 `!`，再后跟宏输入标记树。

## `macro` 是规范的条目

`macro_rules` 宏是按文本限定范围的，并且如果将它视为条目，需要 `#[macro_export]`
（而且还可能需要重导出），但 `macro` 与此不同，因为 `macro` 宏的行为与规范的条目一样。

因此，你可以使用诸如 `pub`、`pub(crate)`、`pub(in path)` 之类的可见性分类符来适当地限定它们。[^proper-item]

[^proper-item]: 译者注：这也意味着，`macro` 宏的导入导出规则符合常规条目。

## 卫生性

到目前为止，卫生性是这两个声明宏系统之间最大的区别。

与具有混合式卫生性 ([mixed site hygiene](./minutiae/hygiene.md)) 的 `macro_rules`
不同，`macro` 具有定义处卫生性 ([definition site hygiene](../proc-macros/hygiene.md))，这意味着
`macro` 不会将标识符泄漏到其调用之外。

这样，下面的代码可以使用 `macro_rules` 宏进行编译，但无法使用 `macro` 定义：

```rust,editable
#![feature(decl_macro)]
// 试着注释下面第一行，然后取消注释下面第二行，看看会发生什么

macro_rules! foo {
// macro foo {
    ($name: ident) => {
        pub struct $name;

        impl $name {
            pub fn new() -> $name {
                $name
            }
        }
    }
}

foo!(Foo);

fn main() {
    // this fails with a `macro`, but succeeds with a `macro_rules`
    let foo = Foo::new();
}
```

未来可能会有计划允许标识符卫生性逃逸 (hygiene bending)。
