# 调试

`rustc` 提供了许多工具来调试通用的语法扩展，并且针对声明宏和过程宏分别量身定制的一些更具体的工具。

有时，语法扩展所展开的内容会有问题，因为你通常看不到展开的的代码。幸运的是， `rustc`
通过不稳定的 `-Zunpretty=expanded` 参数来提供查看展开代码的功能。假设有以下代码：

```rust,ignore
// Shorthand for initializing a `String`.
macro_rules! S {
    ($e:expr) => {String::from($e)};
}

fn main() {
    let world = S!("World");
    println!("Hello, {}!", world);
}
```

使用以下命令编译：

```shell
rustc +nightly -Zunpretty=expanded hello.rs
```

生成以下输出（针对格式进行了修改）：

```rust,ignore
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2018::*;
#[macro_use]
extern crate std;
// Shorthand for initializing a `String`.
macro_rules! S { ($e : expr) => { String :: from($e) } ; }

fn main() {
    let world = String::from("World");
    {
        ::std::io::_print(
            ::core::fmt::Arguments::new_v1(
                &["Hello, ", "!\n"],
                &match (&world,) {
                    (arg0,) => [
                        ::core::fmt::ArgumentV1::new(arg0, ::core::fmt::Display::fmt)
                    ],
                }
            )
        );
    };
}
```

除了 `rustc` 公开了一些方式帮助调试语法扩展之外，对于这里提到的 `-Zunpretty=expanded` 
选项，由 [`dtolnay`](https://github.com/dtolnay) 制作的名为 
[`cargo-expand`](https://github.com/dtolnay/cargo-expand) 的 `cargo` 
插件基本上对它进行了包装，使用起来更加方便。

你也可以使用 [playground](https://play.rust-lang.org/)，点击右上角的 `TOOLS` 按钮来展开语法扩展！
