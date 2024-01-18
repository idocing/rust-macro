# 再谈元变量与宏展开

## 书写宏规则的顺序

一旦语法分析器开始消耗标记以匹配某捕获，整个过程便 **无法停止或回溯** 。
这意味着，无论输入是什么样的，下面这个宏的第二项规则将永远无法被匹配到：

```rust,editable
macro_rules! dead_rule {
    ($e:expr) => { ... };
    ($i:ident +) => { ... };
}

fn main() {
    dead_rule!(x+);
}
```

考虑当以 `dead_rule!(x+)` 形式调用此宏时，将会发生什么。
解析器将从第一条规则开始试图进行匹配：它试图将输入解析为一个表达式。
第一个标记 `x` 作为表达式是有效的，第二个标记——作为二元加符号 `+` 的节点——在表达式中也是有效的。

由此你可能会以为，由于输入中并不包含二元加号 `+` 的右侧元素，
分析器将会放弃尝试这一规则，转而尝试下一条规则。
实则不然：分析器将会 panic 并终止整个编译过程，最终返回一个语法错误。

由于分析器的这一特点，下面这点尤为重要：
一般而言，在书写宏规则时，**应从最具体的开始写起，依次写直到最不具体的** 。

## 片段分类符的跟随限制

为防止将来的语法变动影响宏输入的解析方式，
`macro_rules!` 对紧接元变量后的内容施加了限制。
在 Rust 1.52 中，能够紧跟片段分类符后面的内容具有如下限制[^Follow-set Ambiguity Restrictions]：

* [`stmt`] 和 [`expr`]：`=>`、`,`、`;` 之一
* [`pat`]：`=>`、`,`、`=`、`if`、`in` 之一[^pat-edition]
* [`pat_param`]：`=>`、`,`、`=`、`|`、`if`、`in` 之一
* [`path`] 和 [`ty`]：`=>`、`,`、`=`、`|`、`;`、`:`、`>`、`>>`、`[`、`{`、`as`、`where` 之一；
或者 [`block`] 型的元变量
* [`vis`]：`,`、除了 `priv` 之外的标识符、任何以类型开头的标记、
[`ident`] 或 [`ty`] 或 [`path`] 型的元变量
* 其他片段分类符所跟的内容无限制

[^pat-edition]: 使用 2021 edition 之前的 Rust，`pat` 依然可以跟随 `|`。

反复匹配的情况也遵循这些限制[^Follow-set Ambiguity Restrictions]，也就是说：

1. 如果一个重复操作符（`*` 或 `+`）能让一类元变量重复数次，
那么反复出现的内容就是这类元变量，反复结束之后所接的内容遵照上面的限制。

2. 如果一个重复操作符（`*` 或 `?`）让一类元变量重复零次，
那么元变量之后的内容遵照上面的限制。

[^Follow-set Ambiguity Restrictions]: 内容来自于 *Reference* 
[follow-set-ambiguity-restrictions](https://doc.rust-lang.org/reference/macros-by-example.html#follow-set-ambiguity-restrictions) 一节。

## 编译器拒绝模糊的规则

解析器不会预先运行代码，这意味着如果编译器不能一次就唯一地确定如何解析宏调用，
那么编译器就带着模糊的报错信息而终止运行。
一个触发终止运行的例子是：

```rust,editable
macro_rules! ambiguity {
    ($($i:ident)* $i2:ident) => { };
}

// error:
//    local ambiguity: multiple parsing options: built-in NTs ident ('i') or ident ('i2').
fn main() { ambiguity!(an_identifier); }
```

编译器不会提前看到传入的标识符之后是不是一个 `)`，如果提前看到的话就会解析正确。

## 不基于标记的代换

关于代换元变量 (substitution，这里指把已经进行宏解析的 token 再次传给宏) ，
常常让人惊讶的一面是，尽管 **很像** 是根据标记 (token) 进行代换的，但事实并非如此
——代换基于已经解析的 AST 节点。

思考下面的例子：

```rust,editable
macro_rules! capture_then_match_tokens {
    ($e:expr) => {match_tokens!($e)};
}

macro_rules! match_tokens {
    ($a:tt + $b:tt) => {"got an addition"};
    (($i:ident)) => {"got an identifier"};
    ($($other:tt)*) => {"got something else"};
}

fn main() {
    println!("{}\n{}\n{}\n",
        match_tokens!((caravan)),
        match_tokens!(3 + 6),
        match_tokens!(5));
    println!("{}\n{}\n{}",
        capture_then_match_tokens!((caravan)),
        capture_then_match_tokens!(3 + 6),
        capture_then_match_tokens!(5));
}
```

其结果：

```text
got an identifier
got an addition
got something else

got something else
got something else
got something else
```

通过解析已经传入 AST 节点的输入，代换的结果变得 *很稳定*：你再也无法检查其内容了，
也不再匹配内容。

另一个例子可能也会很令人困惑：

```rust,editable
macro_rules! capture_then_what_is {
    (#[$m:meta]) => {what_is!(#[$m])};
}

macro_rules! what_is {
    (#[no_mangle]) => {"no_mangle attribute"};
    (#[inline]) => {"inline attribute"};
    ($($tts:tt)*) => {concat!("something else (", stringify!($($tts)*), ")")};
}

fn main() {
    println!(
        "{}\n{}\n{}\n{}",
        what_is!(#[no_mangle]),
        what_is!(#[inline]),
        capture_then_what_is!(#[no_mangle]),
        capture_then_what_is!(#[inline]),
    );
}
```

结果是：

```text
no_mangle attribute
inline attribute
something else (#[no_mangle])
something else (#[inline])
```

避免这个意外情况的唯一方式就是使用 [`tt`]、[`ident`] 或者 [`lifetime`] 分类符。
每当你捕获到除此之外的分类符，结果将只能被用于直接输出。
比如这里使用的 `stringify!`[^stringify]，它是一条内置于编译器的语法拓展
（[查看源码可知](https://doc.rust-lang.org/src/core/macros/mod.rs.html#974-978)），
将所有输入标记结合在一起，作为单个字符串输出。


[`item`]:./fragment-specifiers.html#item
[`block`]:./fragment-specifiers.html#block
[`stmt`]:./fragment-specifiers.html#stmt
[`pat`]:./fragment-specifiers.html#pat
[`expr`]:./fragment-specifiers.html#expr
[`ty`]:./fragment-specifiers.html#ty
[`ident`]:./fragment-specifiers.html#ident
[`path`]:./fragment-specifiers.html#path
[`tt`]:./fragment-specifiers.html#tt
[`meta`]:./fragment-specifiers.html#meta
[`lifetime`]:./fragment-specifiers.html#lifetime
[`vis`]:./fragment-specifiers.html#vis
[`literal`]:./fragment-specifiers.html#literal


[^stringify]:这里未包含原作对 `stringify!` 用于替换 (substitution) 场景的 [解读](https://danielkeep.github.io/tlborm/book/mbe-min-captures-and-expansion-redux.html)，因为那个例子的结果有些变化。
