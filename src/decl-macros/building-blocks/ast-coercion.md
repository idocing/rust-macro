# AST 强制转换

在替换 `tt` 时，Rust 的解析器并不十分可靠。
当它期望得到某类特定的语法结构时，
如果摆在它面前的是一坨替换后的 `tt` 标记，就有可能出现问题。
解析器常常直接选择放弃解析，而非尝试去解析它们。
在这类情况中，就要用到 AST 强制转换（简称“强转”）。

```rust,editable
#![allow(dead_code)]

macro_rules! as_expr { ($e:expr) => {$e} }
macro_rules! as_item { ($i:item) => {$i} }
macro_rules! as_pat  { ($p:pat)  => {$p} }
macro_rules! as_stmt { ($s:stmt) => {$s} }
macro_rules! as_ty   { ($t:ty)   => {$t} }

fn main() {
    as_item!{struct Dummy;}

    as_stmt!(let as_pat!(_): as_ty!(_) = as_expr!(42));
}
```

这些强制变换经常与 [下推累积][push-down accumulation] 宏一同使用，
以使解析器能够将最终输出的 `tt` 序列当作某类特定的语法结构来对待。

注意：之所以只有这几种强转宏，
是由宏 **可以展开成什么** 所决定的，
而不是由宏能够捕捉哪些东西所决定的。

[push-down accumulation]: ../patterns/push-down-acc.html
