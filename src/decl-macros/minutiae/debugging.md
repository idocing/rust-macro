# 调试

> 注意：这是一个专门为声明宏量身定做的调试工具清单，调试宏的其他方法可以在语法扩展的
> [调试][syntax-debugging] 章节中找到。

## `trace_macros!`

最有用的是 [`trace_macros!`]，在每次声明宏展开前，它指示编译器记录下声明宏的调用信息。

例如：

```rust,ignore
# // 注意：这需要 nightly Rust
#![feature(trace_macros)]

macro_rules! each_tt {
    () => {};
    ($_tt:tt $($rest:tt)*) => {each_tt!($($rest)*);};
}

each_tt!(foo bar baz quux);
trace_macros!(true);
each_tt!(spim wak plee whum);
trace_macros!(false);
each_tt!(trom qlip winp xod);
#
# fn main() {}
```

输出为：

```text
note: trace_macro
  --> src/main.rs:11:1
   |
11 | each_tt!(spim wak plee whum);
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: expanding `each_tt! { spim wak plee whum }`
   = note: to `each_tt ! (wak plee whum) ;`
   = note: expanding `each_tt! { wak plee whum }`
   = note: to `each_tt ! (plee whum) ;`
   = note: expanding `each_tt! { plee whum }`
   = note: to `each_tt ! (whum) ;`
   = note: expanding `each_tt! { whum }`
   = note: to `each_tt ! () ;`
   = note: expanding `each_tt! {  }`
   = note: to ``
```

它在调试递归很深的宏时尤其有用。

此外，你可以在命令行里，给编译指令附加 `-Z trace-macros` 来打印追踪的宏。

`trace_macros!(false);` 之后的宏不会被这个附加指令追踪到，所以这里会追踪前两个宏。

参考命令：`cargo rustc --bin binary_name  -- -Z trace-macros`

## `log_syntax!`

另一有用的宏是 [`log_syntax!`]。它将使得编译器输出所有经过编译器处理的标记。

比如让编译器“唱首歌”：

```rust
# // 注意：这需要 nightly Rust
#![feature(log_syntax)]

macro_rules! sing {
    () => {};
    ($tt:tt $($rest:tt)*) => {log_syntax!($tt); sing!($($rest)*);};
}

sing! {
    ^ < @ < . @ *
    '\x08' '{' '"' _ # ' '
    - @ '$' && / _ %
    ! ( '\t' @ | = >
    ; '\x08' '\'' + '$' ? '\x7f'
    , # '"' ~ | ) '\x07'
}
#
# fn main() {}
```

比起 `trace_macros!` 来说，它能够做一些更有针对性的调试。

## `macro_railroad` lib

另一个很棒的工具是 [lukaslueg] 编写的 [`macro_railroad`] lib。

它能可视化地生成 Rust `macro_rules!` 宏的语法图 (syntax diagrams)。

它还有一个[浏览器插件][railroad-ext]，和一个可动态可视化声明宏的[静态网页][railroad-demo]。

[syntax-debugging]: ../../syntax-extensions/debugging.md
[`trace_macros!`]: https://doc.rust-lang.org/std/macro.trace_macros.html
[`log_syntax!`]: https://doc.rust-lang.org/std/macro.log_syntax.html
[lukaslueg]: https://github.com/lukaslueg
[`macro_railroad`]: https://github.com/lukaslueg/macro_railroad
[railroad-ext]: https://github.com/lukaslueg/macro_railroad_ext
[railroad-demo]: https://lukaslueg.github.io/macro_railroad_wasm_demo
