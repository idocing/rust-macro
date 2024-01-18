# 源代码解析方式

## 标识化 (Tokenization)

Rust程序编译过程的第一阶段是 [标记解析][tokenization]。
在这一过程中，源代码将被转换成一系列的标记 (token)。

> token：无法被分割的词法单元；在编程语言世界中等价于“单词”。

Rust包含多种标记，比如：

* 标识符 (identifiers): `foo`, `Bambous`, `self`, `we_can_dance`, `LaCaravane`, …
* 字面值 (literals): `42`, `72u32`, `0_______0`, `1.0e-40`, `"ferris was here"`, …
* 关键字 (keywords): `_`, `fn`, `self`, `match`, `yield`, `macro`, …
* 符号   (symbols): `[`, `:`, `::`, `?`, `~`, `@`[^wither-at], …

[等等](https://doc.rust-lang.org/stable/reference/macros-by-example.html#metavariables)。

有些地方值得注意：

1. `self` 既是一个标识符又是一个关键词。
几乎在所有情况下它都被视作是一个关键词，但它有可能被视为标识符。
我们稍后会（骂骂咧咧地）提到这种情况。

2. 关键词里列有一些可疑的家伙，比如 `yield` 和 `macro`。
它们在当前的Rust语言中并没有任何含义，但编译器的确会把它们视作关键词进行解析。
这些词语被保留作语言未来扩充时使用。

3. 符号里也列有一些未被当前语言使用的条目。比如 `<-`，这是历史残留：
目前它被移除了Rust语法，但词法分析器仍然没丢掉它。

4. 注意 `::` 被视作一个独立的标记，而非两个连续的 `:` 。
这一规则适用于截至 Rust 1.2 版本的所有的多字符符号标记。
[^two-lexers]

[^wither-at]: `@` 被用在模式中，用来绑定模式非终止的部分到一个名称——但这似乎被大多数人完全地遗忘了。

[^two-lexers]: 严格来说， Rust 1.52 版本中存在两个词法分析器 (lexer)：
[`rustc_lexer`] 只将单个字符作为 标记 (tokens)；
[`rustc_parse`] 里的 [lexer] 把多个字符作为不同的 标记 (tokens)。

作为对比，某些语言的宏系统正扎根于这一阶段。Rust并非如此。
举例来说，从效果来看，C/C++的宏就是在这里得到处理的。[^lies-damn-lies-cpp]
这也正是下列代码能够运行的原因:
[^cpp-it-seemed-like-a-good-idea-at-the-time]

```c
#define SUB void
#define BEGIN {
#define END }

SUB main() BEGIN
    printf("Oh, the horror!\n");
END
```

[^lies-damn-lies-cpp]: 
实际上，C 预处理程序使用与 C 自身所不同的词法结构，但这些区别很大程度上无关紧要。

[^cpp-it-seemed-like-a-good-idea-at-the-time]: 
是否应该这样运行完全是一个另外的话题了。

## 语法解析 (Parsing)

编译过程的下一个阶段是语法解析 (parsing)。

这一过程中，一系列的 token 将被转换成一棵抽象语法树 (AST: [Abstract Syntax Tree](AST))。
此过程将在内存中建立起程序的语法结构。

举例来说，标记序列 `1+2` 将被转换成某种类似于:

```text
┌─────────┐   ┌─────────┐
│ BinOp   │ ┌╴│ LitInt  │
│ op: Add │ │ │ val: 1  │
│ lhs: ◌  │╶┘ └─────────┘
│ rhs: ◌  │╶┐ ┌─────────┐
└─────────┘ └╴│ LitInt  │
              │ val: 2  │
              └─────────┘
```

AST 将包含 **整个** 程序的结构，但这一结构仅包含词法信息。

举例来讲，在这个阶段编译器虽然可能知道某个表达式提及了某个名为 `a` 的变量，
但它并 **没有办法知道** `a` 究竟是什么，或者它从哪来。

在 AST 生成之后，宏处理过程才开始。
但在讨论宏处理过程之前，我们需要先谈谈标记树 (token tree)。

## 标记树 (Token Trees)

标记树是介于 标记 (token) 与 AST 之间的东西。

首先明确一点，几乎所有标记都构成标记树。
具体来说，它们可被看作标记树叶节点。
还有另一类事物也可被看作标记树叶节点，我们将在稍后提到它。

仅有的一种基础标记不是标记树叶节点——“分组”标记：`(...)`， `[...]` 和 `{...}`。
这三者属于标记树内的节点，正是它们给标记树带来了树状的结构。

给个具体的例子，这列标记：

```text
a + b + (c + d[0]) + e
```

将被解析为这样的标记树：

```text
«a» «+» «b» «+» «(   )» «+» «e»
          ╭────────┴──────────╮
           «c» «+» «d» «[   ]»
                        ╭─┴─╮
                         «0»
```

注意它跟最后生成的 AST 并 *没有关联*。
AST 将仅有一个根节点，而这棵标记树有 *七* 个。
作为参考，最后生成的 AST 应该是这样：

```text
              ┌─────────┐
              │ BinOp   │
              │ op: Add │
            ┌╴│ lhs: ◌  │
┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
│ Var     │╶┘ └─────────┘ └╴│ BinOp   │
│ name: a │                 │ op: Add │
└─────────┘               ┌╴│ lhs: ◌  │
              ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
              │ Var     │╶┘ └─────────┘ └╴│ BinOp   │
              │ name: b │                 │ op: Add │
              └─────────┘               ┌╴│ lhs: ◌  │
                            ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
                            │ BinOp   │╶┘ └─────────┘ └╴│ Var     │
                            │ op: Add │                 │ name: e │
                          ┌╴│ lhs: ◌  │                 └─────────┘
              ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
              │ Var     │╶┘ └─────────┘ └╴│ Index   │
              │ name: c │               ┌╴│ arr: ◌  │
              └─────────┘   ┌─────────┐ │ │ ind: ◌  │╶┐ ┌─────────┐
                            │ Var     │╶┘ └─────────┘ └╴│ LitInt  │
                            │ name: d │                 │ val: 0  │
                            └─────────┘                 └─────────┘
```

理解 AST 与 标记树 (token tree) 之间的区别至关重要。
写宏时，你将同时与这两者打交道。

还有一条需要注意：*不可能* 出现不匹配的小/中/大括号，也不可能存在包含错误嵌套结构的标记树。

[tokenization]: https://en.wikipedia.org/wiki/Lexical_analysis#Tokenization
[reserved]: https://doc.rust-lang.org/reference/keywords.html#reserved-keywords
[`rustc_lexer`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_lexer
[`rustc_parse`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_parse
[lexer]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_parse/src/lexer
[Abstract Syntax Tree]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
