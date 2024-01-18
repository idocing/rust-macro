# 元变量表达式

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/3086-macro-metavar-expr.md)  
> *Tracking Issue*: [rust#83527](https://github.com/rust-lang/rust/issues/83527)  
> *Feature*: `#![feature(macro_metavar_expr)]`  

> 注意：示例代码片段非常简单，只试图展示它们是如何工作的。  
> 关于这些元变量表达式，如果你认为你有合适的、单独使用的小片段，请提交它们！

正如在 [思路][methodical] 中提到的，Rust 有一些特殊的元变量表达式（以下简称表达式）：transcriber[^transcribe] 
可以使用这些表达式来获取有关元变量的信息。如果没有这些表达式，它们所提供的信息就很难甚至不可能获得。

本章将结合用例对它们进行更深入的介绍。

* [`$$`](#dollar-dollar-)
* [`${count(ident, depth)}`](#countident-depth)
* [`${index(depth)}`](#indexdepth)
* [`${length(depth)}`](#lengthdepth)
* [`${ignore(ident)}`](#ignoreident)

[methodical]: ../macros-methodical.md
[^transcribe]: 译者注：在专业的讨论中，尤其涉及元变量表达式，常用 transcribe(r) 一词而不使用 expand (expansion)。

## Dollar Dollar (`$$`)

`$$` 表达式展开为单个 `$`，实际上使其成为转义的 `$`。这让声明宏宏生成新的声明宏。

因为以前的声明宏将无法转义 `$`，所以无法使用元变量、重复和元变量表达式。例如以下代码片段中不使用 `$$`，就无法使用 `bar!`：

```rust
macro_rules! foo {
    () => {
        macro_rules! bar {
            ( $( $any:tt )* ) => { $( $any )* };
            // ^^^^^^^^^^^ error: attempted to repeat an expression containing no syntax variables matched as repeating at this depth
        }
    };
}

foo!();
# fn main() {}
```

问题很明显， `foo!` 的 transcriber 看到有反复捕获的意图，并试图反复捕获，但它的作用域中没有 `$any`
元变量，这导致它产生错误。有了 `$$`，我们就可以解决这个问题[^foo-bar]，因为 `foo` 的 transcriber 不再尝试反复捕获。

```rust
#![feature(macro_metavar_expr)]

macro_rules! foo {
    () => {
        macro_rules! bar {
            ( $$( $$any:tt )* ) => { $$( $$any )* };
        }
    };
}

foo!();
bar!();
# fn main() {}
```

[^foo-bar]: 译者注：在没有 `$$` 之前，存在一种技巧绕过这里的问题：你可以使用 `$tt` 捕获 `$` 来进行转义，比如[这样][tt-$]。

[tt-$]: https://play.rust-lang.org/?version=nightly&mode=debug&edition=2021&gist=9ce18fc79ce17c77d20e74f3c46ee13c

## `count(ident, depth)`

`count` 表达式展开成元变量 `$ident` 在给定反复深度的反复次数。

* `ident` 参数必须是规则作用域中声明的元变量
* `depth ` 参数必须是值小于或等于元变量 `$ident` 出现的最大反复深度的整型字面值
* `count(ident, depth)` 展开成不带后缀的整型字面值标记
* `count(ident)` 是 `count(ident, 0)` 的简写

```rust,editable
#![feature(macro_metavar_expr)]

macro_rules! foo {
    ( $( $outer:ident ( $( $inner:ident ),* ) ; )* ) => {
        println!("count(outer, 0): $outer repeats {} times", ${count(outer)});
        println!("count(inner, 0): The $inner repetition repeats {} times in the outer repetition", ${count(inner, 0)});
        println!("count(inner, 1): $inner repeats {} times in the inner repetitions", ${count(inner, 1)});
    };
}

fn main() {
    foo! {
        outer () ;
        outer ( inner , inner ) ;
        outer () ;
        outer ( inner ) ;
    };
}
```

## `index(depth)`

`index(depth)` 表达式展开为给定反复深度下，当前的迭代索引。

* `depth` 参数表明在第几层反复，这个数字从最内层反复调用表达式开始向外计算
* `index(depth)` 展开成不带后缀的整型字面值标记
* `index()` 是 `index(0)` 的简写

```rust,editable
#![feature(macro_metavar_expr)]

macro_rules! attach_iteration_counts {
    ( $( ( $( $inner:ident ),* ) ; )* ) => {
        ( $(
            $((
                stringify!($inner),
                ${index(1)}, // 这指的是外层反复
                ${index()}  // 这指的是内层反复，等价于 `index(0)`
            ),)*
        )* )
    };
}

fn main() {
    let v = attach_iteration_counts! {
        ( hello ) ;
        ( indices , of ) ;
        () ;
        ( these, repetitions ) ;
    };
    println!("{v:?}");
}
```

## `length(depth)`

`length(depth)` 表达式展开为在给定反复深度的迭代次数。

* `depth` 参数表示在第几层反复，这个数字从最内层反复调用表达式开始向外计算
* `length(depth)` 展开成不带后缀的整型字面值标记
* `length()` 是 `length(0)` 的简写

```rust,editable
#![feature(macro_metavar_expr)]

macro_rules! lets_count {
    ( $( $outer:ident ( $( $inner:ident ),* ) ; )* ) => {
        $(
            $(
                println!(
                    "'{}' in inner iteration {}/{} with '{}' in outer iteration {}/{} ",
                    stringify!($inner), ${index()}, ${length()},
                    stringify!($outer), ${index(1)}, ${length(1)},
                );
            )*
        )*
    };
}

fn main() {
    lets_count!(
        many (small , things) ;
        none () ;
        exactly ( one ) ;
    );
}
```

## `ignore(ident)`

`ignore(ident)` 表达式展开为空，这使得在无需实际展开元变量的时候，像元变量反复展开相同次数的某些内容。

* `ident` 参数必须是规则作用域中声明的元变量

```rust,editable
#![feature(macro_metavar_expr)]

macro_rules! repetition_tuples {
    ( $( ( $( $inner:ident ),* ) ; )* ) => {
        ($(
            $(
                (
                    ${index()},
                    ${index(1)}
                    ${ignore(inner)} // without this metavariable expression, compilation would fail
                ),
            )*
        )*)
    };
}

fn main() {
    let tuple = repetition_tuples!(
        ( one, two ) ;
        () ;
        ( one ) ;
        ( one, two, three ) ;
    );
    println!("{tuple:?}");
}
```
