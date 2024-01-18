# 思路介绍

这一节会介绍 Rust 的[声明宏系统][mbe]，解释该系统如何作为整体运作。

首先会深入构造语法及其关键部分，然后介绍你至少应该了解的通用信息。

[mbe]: https://doc.rust-lang.org/reference/macros-by-example.html

# `macro_rules!`

有了前述知识，我们终于可以介绍 `macro_rules!` 了。如前所述，`macro_rules!`
本身就是一个语法扩展，也就是从技术上说，它并不是 Rust 语法的一部分。它的形式如下：

```rust,ignore
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}
```

**至少得有一条规则**，而且最后一条规则后面的分号可被省略。规则里你可以使用大/中/小括号：
`{}`、`[]`、`()`[^braces]。每条“规则”都形如：

```ignore
    ($matcher) => {$expansion}
```

[^braces]: 译者注：它们的英文名称有时候很重要，因为如果你不认识英文名称的话，会比较难读懂文档（比如
           [`syn`]）。braces `{}`、brackets `[]`、parentheses `()`。

[`syn`]: https://docs.rs/syn/latest/syn/token/index.html#parsing

分组符号可以是任意一种括号，但处于习惯，在模式匹配 (matcher) 外侧使用小括号、展开 
(expansion 也可以叫做 transcriber) 外侧使用大括号。

注意：在规则里选择哪种括号并不会影响宏调用。

而且，实际上，你也可以在调用宏时使用这三种中任意一种括号，只不过使用 `{ ... }` 或者 `( ... );` 
的话会有所不同（关注点在于末尾跟随的分号 `;` ）。有末尾分号的宏调用**总是**会被解析成一个条目 (item)。

如果你好奇的话，`macro_rules!` 的调用将被展开成什么？答案是：空 (nothing)。至少，在 AST
中它被展开为空。它所影响的是编译器内部的结构，以将该宏注册 (register) 
进去。因此，技术上讲你可以在任何一个空展开合法的位置使用 `macro_rules!`。

> 译者注：这里提到两种情况，定义声明宏和使用（或者说调用）声明宏。而且，在括号的选取上：
> 1. 定义的规则不关心 `($matcher) => {$expansion}` 中的**外层**括号类型，但 matcher 和 expansion
>    之内的括号属于匹配和展开的内容，所以它们内部使用什么括号取决于你需要什么语法。
> 2. 假如使用 `m!` 这个宏，如果该宏展开成条目，则必须使用 `m! { ... }` 或者 `m!( ... );`；
>    如果该宏展开成表达式，你可以使用 `m! { ... }` 或者 `m!( ... )` 或者 `m![ ... ]`。
> 3. 实际上，定义宏的括号遵循习惯就好，而使用宏的括号用错的话，只需仔细阅读编译器给你的错误信息，和以上第
>    2 点，就知道怎么改了。

## 匹配

当一个宏被调用时，`macro_rules!` 解释器将按照声明顺序一一检查规则。

对每条规则，它都将尝试将输入标记树的内容与该规则的 `matcher` 进行匹配。某个 matcher [^matcher]
必须与输入**完全**匹配才被认为是一次匹配。

[^matcher]: 译者注：为了简单起见，我不翻译 matcher 这个术语，它指的是被匹配的部分，也就是声明宏规则的前半段。

如果输入与某个 matcher 相匹配，则该调用将替换成相应的展开内容 (`expansion`) ；否则，将尝试匹配下条规则。

如果所有规则均匹配失败，则宏展开失败并报错。

最简单的例子是空 matcher：

```rust,ignore
macro_rules! four {
    () => { 1 + 3 };
}
```

当且仅当匹配到空的输入时，匹配成功，即 `four!()`、`four![]` 或 `four!{}` 三种方式调用是匹配成功的 。

注意所用的分组标记并**不需要**匹配定义时采用的分组标记，因为实际上分组标记并未传给调用。

也就是说，你可以通过 `four![]` 调用上述宏，此调用仍将被视作匹配成功。只有输入的内容才会被纳入匹配考量范围。

matcher 中也可以包含字面上[^literal]的标记树，这些标记树必须被完全匹配。将整个对应标记树在相应位置写下即可。

比如，要匹配标记序列 `4 fn ['spang "whammo"] @_@` ，我们可以这样写：

```rust,ignore
macro_rules! gibberish {
    (4 fn ['spang "whammo"] @_@) => {...};
}
```

使用 `gibberish!(4 fn ['spang "whammo"] @_@])` 即可成功匹配和调用。

你能写出什么标记树，就可以使用什么标记树。

[^literal]: 译者注：这里不是指 Rust 的“字面值”，而是指不考虑含义的标记，比如这个例子中 `fn` 和 `[]`都不是 
            Rust 的 [literal] 标记 ([token])，而是 [keyword] 和 [delimiter] 
            标记，或者从下面谈到的元变量角度看，它们**可以**被 `ident` 或者 `tt` 分类符捕获。

[literal]: https://doc.rust-lang.org/reference/tokens.html#literals
[token]: https://doc.rust-lang.org/reference/tokens.html
[delimiter]: https://doc.rust-lang.org/reference/tokens.html#delimiters
[keyword]: https://doc.rust-lang.org/reference/keywords.html

## 元变量

matcher 还可以包含捕获 (captures)。即基于某种通用语法类别来匹配输入，并将结果捕获到元变量 (metavariable)
中，然后将替换元变量到输出。

捕获的书写方式是：先写美元符号 `$`，然后跟一个标识符，然后是冒号 `:`，最后是捕获方式，比如 `$e:expr`。

捕获方式又被称作“片段分类符” ([fragment-specifier])，必须是以下一种：

* [`block`](./minutiae/fragment-specifiers.md#block)：一个块（比如一块语句或者由大括号包围的一个表达式）
* [`expr`](./minutiae/fragment-specifiers.md#expr)：一个表达式 (expression)
* [`ident`](./minutiae/fragment-specifiers.md#ident)：一个标识符 (identifier)，包括关键字 (keywords)
* [`item`](./minutiae/fragment-specifiers.md#item)：一个条目（比如函数、结构体、模块、`impl` 块）
* [`lifetime`](./minutiae/fragment-specifiers.md#lifetime)：一个生命周期注解（比如 `'foo`、`'static`）
* [`literal`](./minutiae/fragment-specifiers.md#literal)：一个字面值（比如 `"Hello World!"`、`3.14`、`'🦀'`）
* [`meta`](./minutiae/fragment-specifiers.md#meta)：一个元信息（比如 `#[...]` 和 `#![...]` 属性内部的东西）
* [`pat`](./minutiae/fragment-specifiers.md#pat)：一个模式 (pattern)
* [`path`](./minutiae/fragment-specifiers.md#path)：一条路径（比如 `foo`、`::std::mem::replace`、`transmute::<_, int>`）
* [`stmt`](./minutiae/fragment-specifiers.md#stmt)：一条语句 (statement)
* [`tt`](./minutiae/fragment-specifiers.md#tt)：单棵标记树
* [`ty`](./minutiae/fragment-specifiers.md#ty)：一个类型
* [`vis`](./minutiae/fragment-specifiers.md#vis)：一个可能为空的可视标识符（比如 `pub`、`pub(in crate)`）

[fragment-specifier]: https://doc.rust-lang.org/nightly/reference/macros-by-example.html#metavariables

关于片段分类符更深入的描述请阅读本书的[片段分类符](./minutiae/fragment-specifiers.md)一章。

比如以下声明宏捕获一个表达式输入到元变量 `$e`：

```rust,ignore
macro_rules! one_expression {
    ($e:expr) => {...};
}
```

元变量对 Rust 编译器的解析器产生影响，而解析器也会确保元变量总是被“正确无误”地解析。

`expr` 元变量总是捕获完整且符合 Rust 编译版本的表达式。

你可以在有限的情况下同时结合字面上的标记树和元变量。（见 [Metavariables and Expansion Redux] 一节）

当元变量已经在 matcher 中确定之后，你只需要写 `$name` 就能引用元变量。比如：

```rust,ignore
macro_rules! times_five {
    ($e:expr) => { 5 * $e };
}
```

元变量被替换成完整的 AST 节点，这很像宏展开。这也意味着被 `$e` 
捕获的任何标记序列都会被解析成单个完整的表达式。

你也可以一个 matcher 中捕获多个元变量：

```rust,ignore
macro_rules! multiply_add {
    ($a:expr, $b:expr, $c:expr) => { $a * ($b + $c) };
}
```

然后在 expansion 中使用任意次数的元变量：

```rust,ignore
macro_rules! discard {
    ($e:expr) => {};
}
macro_rules! repeat {
    ($e:expr) => { $e; $e; $e; };
}
```

有一个特殊的元变量叫做 [`$crate`] ，它用来指代当前 crate 。

[Metavariables and Expansion Redux]: ./minutiae/metavar-and-expansion.md
[`$crate`]: ./minutiae/hygiene.md#crate

## 反复

matcher 可以有反复捕获 (repetition)，这使得匹配一连串标记 (token)
成为可能。反复捕获的一般形式为 `$ ( ... ) sep rep`。

* `$` 是字面上的美元符号标记
* `( ... )` 是被反复匹配的模式，由小括号包围。
* `sep` 是**可选**的分隔标记。它不能是括号或者反复操作符 `rep`。常用例子有 `,` 和 `;` 。
* `rep` 是**必须**的重复操作符。当前可以是：
    * `?`：表示最多一次重复，所以此时不能前跟分隔标记。
    * `*`：表示零次或多次重复。
    * `+`：表示一次或多次重复。

反复捕获中可以包含任意其他的有效 matcher，比如字面上的标记树、元变量以及任意嵌套的反复捕获。

在 expansion 中，使用被反复捕获的内容时，也采用相同的语法。而且被反复捕获的元变量只能存在于反复语法内。

举例来说，下面这个宏将每一个元素转换成字符串：它先匹配零或多个由逗号分隔的表达式，并分别将它们构造成
`Vec` 的表达式。

```rust,editable
macro_rules! vec_strs {
    (
        // 开始反复捕获
        $(
            // 每个反复必须包含一个表达式
            $element:expr
        )
        // 由逗号分隔
        ,
        // 0 或多次
        *
    ) => {
        // 在这个块内用大括号括起来，然后在里面写多条语句
        {
            let mut v = Vec::new();

            // 开始反复捕获
            $(
                // 每个反复会展开成下面表达式，其中 $element 被换成相应被捕获的表达式
                v.push(format!("{}", $element));
            )*

            v
        }
    };
}

fn main() {
    let s = vec_strs![1, "a", true, 3.14159f32];
    assert_eq!(s, &["1", "a", "true", "3.14159"]);
}
```

你可以在一个反复语句里面使用多次和多个元变量，只要这些元变量以相同的次数重复。所以下面的宏代码正常运行：

```rust,editable
macro_rules! repeat_two {
    ($($i:ident)*, $($i2:ident)*) => {
        $( let $i: (); let $i2: (); )*
    }
}

fn main () {
    repeat_two!( a b c d e f, u v w x y z );
}
```

但是这下面的不能运行：

```rust,editable
# macro_rules! repeat_two {
#     ($($i:ident)*, $($i2:ident)*) => {
#         $( let $i: (); let $i2: (); )*
#     }
# }

fn main() {
    repeat_two!( a b c d e f, x y z );
}
```

运行报以下错误：

```
error: meta-variable `i` repeats 6 times, but `i2` repeats 3 times
 --> src/main.rs:6:10
  |
6 |         $( let $i: (); let $i2: (); )*
  |          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

## 元变量表达式

> *RFC*: [rfcs#1584](https://github.com/rust-lang/rfcs/blob/master/text/3086-macro-metavar-expr.md)
>
> *Tracking Issue*: [rust#83527](https://github.com/rust-lang/rust/issues/83527)
>
> *Feature*: `#![feature(macro_metavar_expr)]`

transcriber[^transcriber] 可以包含所谓的元变量表达 (metavariable expressions)。

元变量表达式为 transcriber 提供了关于元变量的信息 —— 这些信息是不容易获得的。

目前除了 `$$` 表达式外，它们的一般形式都是 `$ { op(...) }`：即除了 `$$` 以外的所有元变量表达式都涉及反复。

可以使用以下表达式（其中 `ident` 是所绑定的元变量的名称，而 `depth` 是整型字面值）：

* `${count(ident)}`：最里层反复 `$ident` 的总次数，相当于 `${count(ident, 0)}`
* `${count(ident，depth)}`：第 `depth` 层反复 `$ident` 的次数
* `${index()}`：最里层反复的当前反复的索引，相当于 `${index(0)}`
* `${index(depth)}`：在第 `depth` 层处当前反复的索引，向外计数
* `${length()}`：最里层反复的重复次数，相当于 `${length(0)}`
* `${length(depth)}`：在第 `depth` 层反复的次数，向外计数
* `${ignore(ident)}`：绑定 `$ident` 进行重复，并展开成空
* `$$`：展开为单个 `$`，这会有效地转义 `$` 标记，因此它不会被展开（转写）

[^transcriber]: 即 expansion，指展开的部分，是每条声明宏规则的后半段。

---

&nbsp;

想了解完整的定义语法，可以参考 Rust Reference 书的 [Macros By Example][mbe] 一章。

