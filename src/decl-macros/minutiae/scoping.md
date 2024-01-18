# 作用域

> 这部分最新的内容可参考 *Reference* 的 
[scoping-exporting-and-importing](https://doc.rust-lang.org/nightly/reference/macros-by-example.html#scoping-exporting-and-importing) 
一节。部分翻译内容引自 *Reference* 
[中文版](https://minstrel1977.gitee.io/rust-reference/macros-by-example.html#scoping-exporting-and-importing)。

函数式宏的作用域规则可能有一点反直觉。（函数式宏包括声明宏与函数式过程宏。）
由于历史原因，宏的作用域并不完全像各种程序项那样工作。

有两种形式的作用域：文本作用域 (textual scope) 和 基于路径的作用域 (path-based scope)。

文本作用域：基于宏在源文件中（定义和使用所）出现的顺序，或是跨多个源文件出现的顺序，
文本作用域是默认的作用域。

基于路径的作用域：与其他程序项作用域的运行方式相同。

当声明宏被 非限定标识符（unqualified identifier，非多段路径段组成的限定性路径）调用时，
会首先在文本作用域中查找。
如果文本作用域中没有任何结果，则继续在基于路径的作用域中查找。

如果宏的名称由路径限定 (qualified with a path) ，则只在基于路径的作用域中查找。

## 文本作用域

### 宏在子模块中可见

与 Rust 语言其余所有部分都不同的是，函数式宏在子模块中仍然可见。

```rust,editable
macro_rules! X { () => {}; }
mod a {
    X!(); // defined
}
mod b {
    X!(); // defined
}
mod c {
    X!(); // defined
}
# fn main() {}
```

> 注意：即使子模组的内容处在不同文件中，这些例子中所述的行为仍然保持不变。

### 宏在定义之后可见

同样与 Rust 语言其余所有部分不同，宏只有在其定义 **之后** 可见。
下例展示了这一点。同时注意到，它也展示了宏不会“漏出” (leak) 其定义所在的作用域：

```rust,editable
mod a {
    // X!(); // undefined
}
mod b {
    // X!(); // undefined
    macro_rules! X { () => {}; }
    X!(); // defined
}
mod c {
    // X!(); // undefined
}
# fn main() {}
```

要清楚，即使你把宏移动到外层作用域，词法依赖顺序的规则依然适用。

```rust,editable
mod a {
    // X!(); // undefined
}

macro_rules! X { () => {}; }

mod b {
    X!(); // defined
}
mod c {
    X!(); // defined
}
# fn main() {}
```

### 宏与宏之间顺序无关 

然而对于宏自身来说，这种具有顺序的依赖行为不存在。
即被调用的宏可以先于调用宏之前声明：

```rust,editable
mod a {
    // X!(); // undefined
}

macro_rules! X { () => { Y!(); }; } // 注意这里的代码运行通过

mod b {
    // 注意这里 X 虽然被定义，但是 Y 不被定义，所以不能使用 X
    // X!(); // defined, but Y! is undefined 
}

macro_rules! Y { () => {}; }

mod c {
    X!(); // defined, and so is Y!
}
# fn main() {}
```

### 宏可以被暂时覆盖

允许多次定义 `macro_rules!` 宏，最后声明的宏会简单地覆盖 (shadow) 上一个声明的同名宏；
如果最后声明的宏离开作用域，上一个宏在有效的作用域内还能被使用。

```rust,editable
macro_rules! X { (1) => {}; }
X!(1);
macro_rules! X { (2) => {}; }
// X!(1); // Error: no rule matches `1`
X!(2);

mod a {
    macro_rules! X { (3) => {}; }
    // X!(2); // Error: no rule matches `2`
    X!(3);
}
// X!(3); // Error: no rule matches `3`
X!(2);

fn main() { }
```
### `#[macro_use]` 属性

这个属性放置在宏定义所在的模块前 或者 `extern crate` 语句前。

1. 在模块前加上 `#[macro_use]` 属性：导出该模块内的所有宏，
从而让导出的宏在所定义的模块结束之后依然可用。

```rust,editable
mod a {
    // X!(); // undefined
}

#[macro_use]
mod b {
    macro_rules! X { () => {}; }
    X!(); // defined
}

mod c {
    X!(); // defined
}
# fn main() {}
```

注意，这可能会产生一些奇怪的后果，因为宏（包括过程宏）中的标识符只有在宏展开的过程中才会被解析。

```rust,editable
mod a {
    // X!(); // undefined
}

#[macro_use]
mod b {
    macro_rules! X { () => { Y!(); }; }
    // X!(); // defined, but Y! is undefined
}

macro_rules! Y { () => {}; }

mod c {
    X!(); // defined, and so is Y!
}
# fn main() {}
```

2. 给 `extern crate` 语句加上 `#[macro_use]` 属性：
把外部 crate 定义且导出的宏引入当前 crate 的根/顶层模块。（当前 crate 使用外部 crate）

假设在外部名称为 `mac` 的 crate 中定义了 `X!` 宏，在当前模块：

```rust,ignore
//// 这里的 `X!` 与 `Y!` 无关，前者定义于外部 crate，后者定义于当前 crate

mod a {
    // X!(); // defined, but Y! is undefined
}

macro_rules! Y { () => {}; }

mod b {
    X!(); // defined, and so is Y!
}

#[macro_use] extern crate macs;
mod c {
    X!(); // defined, and so is Y!
}

# fn main() {}
```

### 当宏放在函数内

前四条作用域规则同样适用于函数。
至于第五条规则， `#[macro_use]` 属性并不直接作用于函数。

```rust,editable
macro_rules! X {
    () => { Y!() };
}

fn a() {
    macro_rules! Y { () => {"Hi!"} }
    assert_eq!(X!(), "Hi!");
    {
        assert_eq!(X!(), "Hi!");
        macro_rules! Y { () => {"Bye!"} }
        assert_eq!(X!(), "Bye!");
    }
    assert_eq!(X!(), "Hi!");
}

fn b() {
    macro_rules! Y { () => {"One more"} }
    assert_eq!(X!(), "One more");
}
# 
# fn main() {
#     a();
#     b();
# }
```

### 关于宏声明的位置

由于前述种种规则，一般来说，
建议将所有应对整个 `crate` 均可见的宏的定义置于根模块的最顶部，
借以确保它们 **一直** 可用。
这个建议和适用于在文件 `mod` 定义的宏：

```rust,ignore
#[macro_use]
mod some_mod_that_defines_macros;
mod some_mod_that_uses_those_macros;
```

这里的顺序很重要，因为第二个模块依赖于第一个模块的宏，
所以改变这两个模块的顺序会无法编译。

## 基于路径的作用域

Rust 的 `macro_rules!` 宏 默认并没有基于路径的作用域。

然而，如果这个宏被加上 `#[macro_export]` 属性，那么它就在 crate 的根作用域里被定义，
而且能直接使用它。

[导入/导出宏][Import and Export] 一章会更深入地探讨这个属性。


[Import and Export]: ./import-export.html

