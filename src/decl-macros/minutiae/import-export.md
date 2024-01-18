# 导入/导出

在 Rust 的 2015 和 2018 版本中，导入 `macro_rules!` 宏是不一样的。
仍然建议阅读这两部分，因为 2018 版使用的结构在 2015 版中做出了解释。

## 2015 版本

### `#[macro_use]`

[作用域][scoping chapter] 一章中介绍的 `#[macro_use]` 属性
适用于模块或者 `external crates` 。例如：

```rust
#[macro_use]
mod macros {
    macro_rules! X { () => { Y!(); } }
    macro_rules! Y { () => {} }
}

X!();
#
# fn main() {}
```

### `#[macro_export]`

可通过 `#[macro_export]` 将宏从当前`crate`导出。注意，这种方式 **无视** 所有可见性设定。

定义 lib 包 `macs` 如下：

```rust,ignore
mod macros {
    #[macro_export] macro_rules! X { () => { Y!(); } }
    #[macro_export] macro_rules! Y { () => {} }
}

// X! 和 Y! 并非在此处定义的，但它们 **的确** 被导出了（在此处可用）
// 即便 `macros` 模块是私有的
```

下面（在使用 `macs` lib 的 crate 中）的代码会正常工作：

```rust,ignore
X!(); // X 在当前 crate 中被定义
#[macro_use] extern crate macs; // 从 `macs` 中导入 X
X!(); // 这里的 X 是最新声明的 X，即 `macs` crate 中导入的 X
# 
# fn main() {}
```

正如 [作用域][scoping chapter] 一章所说，`#[macro_use]` 作用于 `extern crate` 时，
会强制把导出的宏提到 crate 的顶层模块（根模块），所以这里无须使用 `macs::macros` 路径。

> 注意：只有在根模组中，才可将 `#[macro_use]` 用于 `extern crate`。

在从 `extern crate` 导入宏时，可显式控制导入 **哪些** 宏。
从而利用这一特性来限制命名空间污染，或是覆盖某些特定的宏。就像这样：

```rust,ignore
// 只导入 `X!` 这一个宏
#[macro_use(X)] extern crate macs;

// X!(); // X! 已被定义，但 Y! 未被定义。X 与 Y 无关系。

macro_rules! Y { () => {} }

X!(); // X 和 Y 都被定义

fn main() {}
```

当导出宏时，常常出现的情况是，宏定义需要其引用所在 `crate` 内的非宏符号。
由于 `crate` 可能被重命名等，我们可以使用一个特殊的替换变量 [`$crate`] 。
它总将被扩展为宏定义所在的 `crate` 的绝对路径（比如 `:: macs` ）。

如果你的编译器版本小于 1.30（即 2018 版之前），那么这招并不适用于宏。
也就是说，你没办法采用类似 `$crate::Y!` 的代码来引用自己 `crate` 里的定义的宏。
这表示结合 `#[macro_use]` 来选择性导入会无法保证某个名称的宏在另一个 crate 导入同名宏时依然可用。

推荐的做法是，在引用非宏名称时，总是采用绝对路径。
这样可以最大程度上避免冲突，包括跟标准库中名称的冲突。

[`$crate`]:./hygiene.html#crate-元变量

## 2018 版本

2018 版本让使用 `macro_rules!` 宏更简单。
因为新版本设法让 Rust 中某些特殊的东西更像正常的 items 。
这意味着我们能以命名空间的方式正确导入和使用宏！

因此，不必使用 `#[macro_use]` 来导入 来自 extern crate 导出的宏 到全局命名空间，
现在我们这样做就好了：

```rust,ignore
use some_crate::some_macro;

fn main() {
    some_macro!("hello");
    // as well as
    some_crate::some_other_macro!("macro world");
}
```

可惜，这只适用于导入外部 crate 的宏；
如果你使用在自己 crate 定义的 `macro_rules!` 宏，
那么依然需要把 `#[macro_use]` 添加到宏所定义的模块上来引入模块里面的宏。
因而 [作用域规则][scoping chapter] 就像之前谈论的那样生效。

> [`$crate`] 前缀（元变量）在 2018 版中可适用于任何东西，
> 在 1.31 版之后，宏 和类似 item 的东西都能用 `$crate` 导入了。

[scoping chapter]:./scoping.html
