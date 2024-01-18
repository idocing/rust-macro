# 解析 Rust

在有些情况下解析某些 Rust [items] 会很有用。
这一章会展示一些能够解析 Rust 中更复杂的 items 的宏。

[items]:https://doc.rust-lang.org/nightly/reference/items.html

[transcribers]:https://doc.rust-lang.org/nightly/reference/macros-by-example.html

这些宏目的不是解析整个 items 语法，而是解析通用、有用的部分，
解析的方式也不会太复杂。
也就是说，我们不会涉及解析 *泛型* 之类的东西。

重点在于宏的匹配方式 (matchers) ；展开的部分 （ *Reference* 里使用的术语叫做 [transcribers] ），
仅仅用作例子，不需要特别关心它。

## 函数

```rust,editable
macro_rules! function_item_matcher {
    (

        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis fn $name:ident ( $( $arg_name:ident : $arg_ty:ty ),* $(,)? )
    //                          ^~~~~~~~~~~~~~~~argument list!~~~~~~~~~~~~~~^
            $( -> $ret_ty:ty )?
    //      ^~~~return type~~~^
            { $($tt:tt)* }
    //      ^~~~~body~~~~^
    ) => {
        $( #[$meta] )*
        $vis fn $name ( $( $arg_name : $arg_ty ),* ) $( -> $ret_ty )? { $($tt)* }
    }
}

#function_item_matcher!(
#    #[inline]
#    #[cold]
#    pub fn foo(bar: i32, baz: i32, ) -> String {
#        format!("{} {}", bar, baz)
#    }
#);
#
# fn main() {
#     assert_eq!(foo(13, 37), "13 37");
# }
```

这是一个简单的匹配函数的例子，
传入宏的函数不能包含 `unsafe`、`async`、泛型和 where 语句。
如果需要解析这些内容，则最好使用 `proc-macro` （过程宏） 代替。

这个例子可以检查函数签名，从中生成一些额外的东西，
然后再重新返回 (re-emit) 整个函数。
有点像 `Derive` 过程宏，虽然功能没那么强大，但是是为函数服务的
（ `Derive` 不作用于函数）。

> 理想情况下，我们对参数捕获宁愿使用 `pat` 分类符，而不是 `ident` 分类符，
但这里目前不被允许（因为前者的跟随限制，不允许其后使用 `:` ）。
幸好在函数签名里面不常使用模式 ( `pat` ) ，所以这个例子还不错。

## 方法

有时我们想解析方法 (methods)，方法就是通过 `self` 的某种形式指向对象的函数。
这让事情变得棘手多了。

> WIP （待完善）

## 结构体

```rust,editable
macro_rules! struct_item_matcher {
    // Unit-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident;
    ) => {
        $( #[$meta] )*
        $vis struct $name;
    };

    // Tuple-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident (
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~~^
                $field_vis:vis $field_ty:ty
    //          ^~~~~~a single field~~~~~~^
            ),*
        $(,)? );
    ) => {
        $( #[$meta] )*
        $vis struct $name (
            $(
                $( #[$field_meta] )*
                $field_vis $field_ty
            ),*
        );
    };

    // Named-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident {
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~!^
                $field_vis:vis $field_name:ident : $field_ty:ty
    //          ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
            ),*
        $(,)? }
    ) => {
        $( #[$meta] )*
        $vis struct $name {
            $(
                $( #[$field_meta] )*
                $field_vis $field_name : $field_ty
            ),*
        }
    }
}

#struct_item_matcher!(
#    #[allow(dead_code)]
#    #[derive(Copy, Clone)]
#    pub(crate) struct Foo { 
#       pub bar: i32,
#       baz: &'static str,
#       qux: f32
#    }
#);
#struct_item_matcher!(
#    #[derive(Copy, Clone)]
#    pub(crate) struct Bar;
#);
#struct_item_matcher!(
#    #[derive(Clone)]
#    pub(crate) struct Baz (i32, pub f32, String);
#);
#fn main() {
#    let _: Foo = Foo { bar: 42, baz: "macros can be nice", qux: 3.14, };
#    let _: Bar = Bar;
#    let _: Baz = Baz(2, 0.1234, String::new());
#}
```

## 枚举体

解析枚举体比解析结构体更复杂一点，所以会用上 [模式][patterns] 这章讨论的技巧：
[`TT` 撕咬机][Incremental TT Muncher] 和 [内用规则][Internal Rules] 。

不是重新构造被解析的枚举体，而是只访问枚举体所有的标记 (tokens)，
因为重构枚举体将需要我们再通过 [下推累积][Push Down Accumulator] 
临时组合所有已解析的标记 (tokens) 。

```rust,editable
macro_rules! enum_item_matcher {
    // tuple variant
    (@variant $variant:ident (
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~~^
            $field_vis:vis $field_ty:ty
    //      ^~~~~~a single field~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    ) $(, $($tt:tt)* )? ) => {

        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*); )?
    };

    // named variant
    (@variant $variant:ident {
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~!^
            $field_vis:vis $field_name:ident : $field_ty:ty
    //      ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    } $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*); )?
    };

    // unit variant
    (@variant $variant:ident $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*); )?
    };

    // trailing comma
    (@variant ,) => {};
    // base case
    (@variant) => {};

    // entry point
    (
        $( #[$meta:meta] )*
        $vis:vis enum $name:ident {
            $($tt:tt)*
        }
    ) => {
        enum_item_matcher!(@variant $($tt)*);
    };
}

enum_item_matcher!(
    #[derive(Copy, Clone)]
    pub(crate) enum Foo {
        Bar,
        Baz,
    }
);
enum_item_matcher!(
    #[derive(Copy, Clone)]
    pub(crate) enum Bar {
        Foo(i32, f32),
        Bar,
        Baz(),
    }
);
enum_item_matcher!(
    #[derive(Clone)]
    pub(crate) enum Baz {}
);

fn main() {}
```

[patterns]:/patterns.html
[Push Down Accumulator]:/patterns/push-down-acc.html
[Internal Rules]:/patterns/internal-rules.html
[Incremental TT Muncher]:/patterns/tt-muncher.html
