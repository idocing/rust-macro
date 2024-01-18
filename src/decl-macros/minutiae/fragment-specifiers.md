# 片段分类符

正如在 [思路](../macros-methodical.md) 一章看到的，截至 1.60 版本， Rust 已有
14 个片段分类符 (Fragment Specifiers，以下简称分类符)[^metavariables]。

这一节会更深入地探讨他们之中的细节，每次都会展示几个匹配的例子。

> 注意：除了 `ident`、`lifetime` 和 `tt` 分类符之外，其余的分类符在匹配后生成的 
> AST 是不清楚的 (opaque)，这使得在之后的宏调用时不可能检查 (inspect) 捕获的结果。[^opaque]

* [`block`](#block)
* [`expr`](#expr)
* [`ident`](#ident)
* [`item`](#item)
* [`lifetime`](#lifetime)
* [`literal`](#literal)
* [`meta`](#meta)
* [`pat`](#pat)
* [`pat_param`](#pat_param)
* [`path`](#path)
* [`stmt`](#stmt)
* [`tt`](#tt)
* [`ty`](#ty)
* [`vis`](#vis)

[^metavariables]: 最新内容可参考 *Reference* 的 [Metavariables](https://doc.rust-lang.org/nightly/reference/macros-by-example.html#metavariables) 一节。
[^opaque]: 推荐通过 rust quiz [#9](https://dtolnay.github.io/rust-quiz/9) 来理解这句话。

## `block`

`block` 分类符只匹配 
[block 表达式](https://doc.rust-lang.org/reference/expressions/block-expr.html)。

块 (block) 由 `{` 开始，接着是一些语句，最后是可选的表达式，然后以 `}` 结束。
块的类型要么是最后的值表达式类型，要么是 `()` 类型。

```rust,editable
macro_rules! blocks {
    ($($block:block)*) => ();
}

blocks! {
    {}
    {
        let zig;
    }
    { 2 }
}
fn main() {}
```

## `expr`

`expr` 分类符用于匹配任何形式的表达式
([expression](https://doc.rust-lang.org/reference/expressions.html))。

（如果把 Rust 视为面向表达式的语言，那么它有很多种表达式。）

```rust,editable
macro_rules! expressions {
    ($($expr:expr)*) => ();
}

expressions! {
    "literal"
    funcall()
    future.await
    break 'foo bar
}
fn main() {}
```

## `ident`

`ident` 分类符用于匹配任何形式的标识符 
([identifier](https://doc.rust-lang.org/reference/identifiers.html)) 或者关键字。
。

```rust,editable
macro_rules! idents {
    ($($ident:ident)*) => ();
}

idents! {
    // _ <- `_` 不是标识符，而是一种模式
    foo
    async
    O_________O
    _____O_____
}
fn main() {}
```

## `item`

`item` 分类符只匹配 Rust 的 [item](https://doc.rust-lang.org/reference/items.html) 
的 **定义** (definitions) ，
不会匹配指向 item 的标识符 (identifiers)。例子：

```rust,editable
macro_rules! items {
    ($($item:item)*) => ();
}

items! {
    struct Foo;
    enum Bar {
        Baz
    }
    impl Foo {}
    /*...*/
}
fn main() {}
```

[`item`](https://doc.rust-lang.org/reference/items.html) 
是在编译时完全确定的，通常在程序执行期间保持固定，并且可以驻留在只读存储器中。具体指：

*   [modules](https://doc.rust-lang.org/reference/items/modules.html)
*   [`extern crate` declarations](https://doc.rust-lang.org/reference/items/extern-crates.html)
*   [`use` declarations](https://doc.rust-lang.org/reference/items/use-declarations.html)
*   [function definitions](https://doc.rust-lang.org/reference/items/functions.html)
*   [type definitions](https://doc.rust-lang.org/reference/items/type-aliases.html)
*   [struct definitions](https://doc.rust-lang.org/reference/items/structs.html)
*   [enumeration definitions](https://doc.rust-lang.org/reference/items/enumerations.html)
*   [union definitions](https://doc.rust-lang.org/reference/items/unions.html)
*   [constant items](https://doc.rust-lang.org/reference/items/constant-items.html)
*   [static items](https://doc.rust-lang.org/reference/items/static-items.html)
*   [trait definitions](https://doc.rust-lang.org/reference/items/traits.html)
*   [implementations](https://doc.rust-lang.org/reference/items/implementations.html)
*   [`extern` blocks](https://doc.rust-lang.org/reference/items/external-blocks.html)

## `lifetime`

`lifetime` 分类符用于匹配生命周期注解或者标签
([lifetime or label](https://doc.rust-lang.org/reference/tokens.html#lifetimes-and-loop-labels))。
它与 [`ident`](#ident) 很像，但是 `lifetime` 会匹配到前缀 `''` 。

```rust,editable
macro_rules! lifetimes {
    ($($lifetime:lifetime)*) => ();
}

lifetimes! {
    'static
    'shiv
    '_
}
fn main() {}
```

## `literal`

`literal` 分类符用于匹配字面表达式 
([literal expression](https://doc.rust-lang.org/reference/expressions/literal-expr.html))。

```rust,editable
macro_rules! literals {
    ($($literal:literal)*) => ();
}

literals! {
    -1
    "hello world"
    2.3
    b'b'
    true
}
fn main() {}
```

## `meta`

`meta` 分类符用于匹配属性 ([attribute](https://doc.rust-lang.org/reference/attributes.html))，
准确地说是属性里面的内容。通常你会在 `#[$meta:meta]` 或 `#![$meta:meta]` 模式匹配中
看到这个分类符。

```rust,editable
macro_rules! metas {
    ($($meta:meta)*) => ();
}

metas! {
    ASimplePath
    super::man
    path = "home"
    foo(bar)
}
fn main() {}
```

> 针对文档注释简单说一句：
> 文档注释其实是具有 `#[doc="…"]` 形式的属性，`...` 实际上就是注释字符串，
> 这意味着你可以在在宏里面操作文档注释！

## `pat`

`pat` 分类符用于匹配任何形式的模式 
([pattern](https://doc.rust-lang.org/reference/patterns.html))，包括 2021 edition
开始的 [or-patterns](https://doc.rust-lang.org/reference/patterns.html#or-patterns)。

```rust,editable,edition2021
macro_rules! patterns {
    ($($pat:pat)*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
    0 | 1 | 2 | 3 
}
fn main() {}
```

## `pat_param`

从 2021 edition 起， or-patterns 模式开始应用，这让 `pat` 分类符不再允许跟随 `|`。

为了避免这个问题或者说恢复旧的 `pat` 分类符行为，你可以使用 `pat_param` 片段，它允许
`|` 跟在它后面，因为 `pat_param` 不允许 top level 或 or-patterns。

```rust,editable,edition2021
macro_rules! patterns {
    (pat: $pat:pat) => {
        println!("pat: {}", stringify!($pat));
    };
    (pat_param: $($pat:pat_param)|+) => {
        $( println!("pat_param: {}", stringify!($pat)); )+
    };
}
fn main() {
    patterns! {
       pat: 0 | 1 | 2 | 3
    }
    patterns! {
       pat_param: 0 | 1 | 2 | 3
    }
}
```

```rust,editable,edition2021
macro_rules! patterns {
    ($( $( $pat:pat_param )|+ )*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
    0 | 1 | 2 | 3 
}
fn main() {}
```

## `path`

`path` 分类符用于匹配类型中的路径
([TypePath](https://doc.rust-lang.org/reference/paths.html#paths-in-types))。

这包括函数式的 trait 形式。

```rust,editable
macro_rules! paths {
    ($($path:path)*) => ();
}

paths! {
    ASimplePath
    ::A::B::C::D
    G::<eneri>::C
    FnMut(u32) -> ()
}
fn main() {}
```

## `stmt`

`stmt` 分类符只匹配的 语句 ([statement](https://doc.rust-lang.org/reference/statements.html))。
除非 item 语句要求结尾有分号，否则 **不会** 匹配语句最后的分号。

什么叫 item 语句要求结尾有分号呢？单元结构体 (Unit-Struct) 就是一个简单的例子，
因为它定义中必须带上结尾的分号。

赶紧用例子展示上面说的是啥意思吧。下面的宏只给出它所捕获的内容，因为有几行不能通过编译。

```rust,editable
macro_rules! statements {
    ($($stmt:stmt)*) => ($($stmt)*);
}

fn main() {
    statements! {
        struct Foo;
        fn foo() {}
        let zig = 3
        let zig = 3;
        3
        3;
        if true {} else {}
        {}
    }
}
```

你可以根据报错内容试着删除不能编译的代码，结合 [`stmt`](#stmt) 小节开头的文字再琢磨琢磨。
你如果正浏览使用 mdbook 渲染的页面，那么可以直接运行和修改这段代码。

虽然源代码编译失败，但是我们可以展开宏[^debugging]，
使用 [playground](https://play.rust-lang.org/) 的
`Expand macros` 工具 (tool)；或者把代码复制到本地，在 nightly Rust 版本中使用
`cargo rustc -- -Zunstable-options --pretty=expanded` 命令得到宏展开结果：

```rust,ignore
# warning: unnecessary trailing semicolon
#   --> src/main.rs:10:20
#    |
# 10 |         let zig = 3;
#    |                    ^ help: remove this semicolon
#    |
#    = note: `#[warn(redundant_semicolons)]` on by default
# 
# warning: unnecessary trailing semicolon
#   --> src/main.rs:12:10
#    |
# 12 |         3;
#    |          ^ help: remove this semicolon
# 
# #![feature(prelude_import)]
# #[prelude_import]
# use std::prelude::rust_2018::*;
# #[macro_use]
# extern crate std;
# macro_rules! statements { ($ ($ stmt : stmt) *) => ($ ($ stmt) *) ; }

fn main() {
    struct Foo;
    fn foo() { }
    let zig = 3;
    let zig = 3;
    ;
    3;
    3;
    ;
    if true { } else { }
    { }
}
```

由此我们知道：

1. 虽然 `stmt` 分类符没有捕获语句末尾的分号，但它依然在所需的时候返回了 (emit) 
语句。原因很简单，分号本身就是有效的语句。所以我们实际输入 10 个语句调用了宏，而不是 8
个！这在把多个反复捕获放入一个反复展开时很重要，因为此时反复的次数必须相同。

2. 在这里你应该注意到：`struct Foo;` 被匹配到了。否则我们会看到像其他情况一样有一个额外 `;` 
语句。由前所述，这能想通：item 语句需要分号，所以这个分号能被匹配到。

3. 仅由块表达式或控制流表达式组成的表达式结尾没有分号，
其余的表达式捕获后产生的表达式会尾随一个分号（在这个例子中，正是这里出错）。

这里提到的细节能在 Reference 的 [statement](https://doc.rust-lang.org/reference/statements.html)
一节中找到。但个细节通常这并不重要，除了要注意反复次数，通常没什么问题。

[^debugging]: 可阅读 [调试](./debugging.md) 一章

## `tt`

`tt` 分类符用于匹配标记树 (TokenTree)。
如果你是新手，对标记树不了解，那么需要回顾本书 
[标记树](../syntax/source-analysys.html#标记树-token-trees)
一节。`tt` 分类符是最有作用的分类符之一，因为它能匹配几乎所有东西，
而且能够让你在使用宏之后检查 (inspect) 匹配的内容。

这让你可以编写非常强大的宏技巧，比如
[tt-muncher](../patterns/tt-muncher.md) 和 
[push-down-accumulator](../patterns/push-down-acc.md)。

## `ty`

`ty` 分类符用于匹配任何形式的类型表达式 ([type expression](https://doc.rust-lang.org/reference/types.html#type-expressions))。

类型表达式是在 Rust 中指代类型的语法。

```rust,editable
macro_rules! types {
    ($($type:ty)*) => ();
}

types! {
    foo::bar
    bool
    [u8]
    impl IntoIterator<Item = u32>
}
fn main() {}
```

## `vis`

`vis` 分类符会匹配 **可能为空** 可见性修饰符 ([Visibility qualifier](https://doc.rust-lang.org/reference/visibility-and-privacy.html))。

```rust,editable
macro_rules! visibilities {
    //         ∨~~注意这个逗号，`vis` 分类符自身不会匹配到逗号
    ($($vis:vis,)*) => ();
}

visibilities! {
    , // 没有 vis 也行，因为 $vis 隐式包含 `?` 的情况
    pub,
    pub(crate),
    pub(in super),
    pub(in some_path),
}
fn main() {}
```

`vis` 实际上只支持例子里的几种方式，因为这里的 visibility 
指的是可见性，与私有性相对。而涉及这方面的内容只有与 `pub` 
的关键字。所以，`vis` 在关心匹配输入的内容是公有还是私有时有用。

此外，如果匹配时，其后没有标记流，整个宏会匹配失败：

```rust,editable
macro_rules! non_optional_vis {
    ($vis:vis) => ();
}
non_optional_vis!();
// ^^^^^^^^^^^^^^^^ error: missing tokens in macro arguments
fn main() {}
```

重点在于“可能为空”。你可能想到这是隐藏了 `?`
重复操作符的分类符，这样你就不用直接在反复匹配时使用 
`?` —— 其实你不能将它和 `?` 一起在重复模式匹配中使用。

可以匹配 `$vis:vis $ident:ident`，但不能匹配 `$(pub)? $ident:ident`，因为 `pub`
表明一个有效的标识符，所以后者是模糊不清的。

```rust,editable
macro_rules! vis_ident {
    ($vis:vis $ident:ident) => ();
}
vis_ident!(pub foo); // this works fine

macro_rules! pub_ident {
    ($(pub)? $ident:ident) => ();
}
pub_ident!(pub foo);
        // ^^^ error: local ambiguity when calling macro `pub_ident`: multiple parsing options: built-in NTs ident ('ident') or 1 other option.
fn main() {}
```

而且，搭配 `tt` 分类符和递归展开去匹配空标记也会导致有趣而奇怪的事情。

当 `pub` 匹配了空标记，元变量依然算一次被捕获，又因为它不是 `tt`、`ident` 或
`lifetime`，所以再次展开时是不清楚的。

这意味着如果这种捕获的结果传递给另一个将它视为 `tt` 的宏调用，你最终得到一棵空的标记树。

```rust,editable
macro_rules! it_is_opaque {
    (()) => { "()" };
    (($tt:tt)) => { concat!("$tt is ", stringify!($tt)) };
    ($vis:vis ,) => { it_is_opaque!( ($vis) ); }
}
fn main() {
    // this prints "$tt is ", as the recursive calls hits the second branch with
    // an empty tt, opposed to matching with the first branch!
    println!("{}", it_is_opaque!(,));
}
```

