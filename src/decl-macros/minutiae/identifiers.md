# 非标识符的“标识符”

> 译者注：这里需要记住的重点是
>
> 1. `self` 可以使用 [`ident`] 或者 [`tt`] 分类符来匹配；
>
> 2. `_` 只能在模式中使用，不能用 [`ident`] 分类符匹配，而是用  [`pat`] 或者
> [`tt`] 分类符匹配。

有两个标记，当你最终撞见时，很有可能认为它们是标识符 ([`ident`])，但实际上它们不是。

然而正是这些标记，在某些情况下又的确是标识符。

## `self`

第一个是 `self`。毫无疑问，它是一个 **关键词** ([keyword])。在一般的 Rust
代码中，不可能出现把它解读成标识符的情况；但在宏中这种情况则有可能发生：

[keyword]: https://doc.rust-lang.org/reference/keywords.html

```rust,editable
macro_rules! what_is {
    (self) => {"the keyword `self`"};
    ($i:ident) => {concat!("the identifier `", stringify!($i), "`")};
}

macro_rules! call_with_ident {
    ($c:ident($i:ident)) => {$c!($i)};
}

fn main() {
    println!("{}", what_is!(self));
    println!("{}", call_with_ident!(what_is(self)));
}
```

上述代码的输出将是：

```text
the keyword `self`
the keyword `self`
```

但这说不通啊！`call_with_ident!` 要求一个标识符，而且它的确匹配到了，还成功替换了！所以，
`self` 同时是一个关键词，但又不是。你可能会想，好吧，但这鬼东西哪里重要呢？看看这个：

```rust,editable
macro_rules! make_mutable {
    ($i:ident) => {let mut $i = $i;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_mutable!(self);
        self.0 *= 2;
        self
    }
}

fn main() {
    println!("{:?}", Dummy(4).double().0);
}
```

编译它会失败，并报错：

```rust,ignore
error: `mut` must be followed by a named binding
 --> src/main.rs:2:24
  |
2 |     ($i:ident) => {let mut $i = $i;};
  |                        ^^^^^^ help: remove the `mut` prefix: `self`
...
9 |         make_mutable!(self);
  |         -------------------- in this macro invocation
  |
  = note: `mut` may be followed by `variable` and `variable @ pattern`
```

所以说，宏在匹配的时候，会欣然把 `self` 当作标识符接受，进而允许你把 `self`
带到那些实际上没办法使用的情况中去。但是，也成吧，既然得同时记住 `self`
既是关键词又是标识符，那下面这个讲道理应该可行，对吧？

```rust,editable
macro_rules! make_self_mutable {
    ($i:ident) => {let mut $i = self;};
}

struct Dummy(i32);

impl Dummy {
    fn double(self) -> Dummy {
        make_self_mutable!(mut_self);
        mut_self.0 *= 2;
        mut_self
    }
}

fn main() {
    println!("{:?}", Dummy(4).double().0);
}
```

实际上也不行，编译错误变成：

```rust,ignore
error[E0424]: expected value, found module `self`
  --> src/main.rs:2:33
   |
2  |       ($i:ident) => {let mut $i = self;};
   |                                   ^^^^ `self` value is a keyword only available in methods with a `self` parameter
...
8  | /     fn double(self) -> Dummy {
9  | |         make_self_mutable!(mut_self);
   | |         ----------------------------- in this macro invocation
10 | |         mut_self.0 *= 2;
11 | |         mut_self
12 | |     }
   | |_____- this function has a `self` parameter, but a macro invocation can only access identifiers it receives from parameters
   |
```

这同样也说不通。这简直就像是在抱怨说，它看见的两个 `self` 不是同一个 `self`
... 就搞得像关键词 `self` 就像标识符一样，也有卫生性。

```rust,editable
macro_rules! double_method {
    ($body:expr) => {
        fn double(mut self) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {{
        self.0 *= 2;
        self
    }}
}

fn main() {
    println!("{:?}", Dummy(4).double().0);
}
```

还是报同样的错。那这个如何：

```rust,editable
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double(mut $self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {self, {
        self.0 *= 2;
        self
    }}
}

fn main() {
    println!("{:?}", Dummy(4).double().0);
}
```

终于管用了。所以说，`self` 是关键词，但如果想它变成标识符，那么同时也能是一个标识符。

那么，相同的道理对类似的其它东西有用吗？

## `_` 

```rust,editable
macro_rules! double_method {
    ($self_:ident, $body:expr) => {
        fn double($self_) -> Dummy {
            $body
        }
    };
}

struct Dummy(i32);

impl Dummy {
    double_method! {_, 0}
}

fn main() {
    println!("{:?}", Dummy(4).double().0);
}
```

```rust,ignore
error: no rules expected the token `_`
  --> src/main.rs:12:21
   |
1  | macro_rules! double_method {
   | -------------------------- when calling this macro
...
12 |     double_method! {_, 0}
   |                     ^ no rules expected this token in macro call
```

哈，当然不行。即便它 **如同** `self` 一样从定义上讲是标识符，但 [`_`][_] 
在模式以及表达式中是一个合法的 (valid) 关键词，而不是一个标识符。

[_]: https://doc.rust-lang.org/reference/patterns.html#wildcard-pattern

---

你可能觉得，既然 `_` 在模式中有效，那换成 `$self_:pat` 是不是就能一石二鸟了呢？

可惜了，也不行，因为 `self` 不是一个有效的模式。

如果你真想同时匹配这两个标记，仅有的办法是换用 `tt` 来匹配。

[`tt`]:./fragment-specifiers.html#tt
[`ident`]:./fragment-specifiers.html#ident
[`pat`]:./fragment-specifiers.html#pat
[片段分类符]:./fragment-specifiers.html
