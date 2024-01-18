# `TT` 捆绑

```rust,editable
macro_rules! call_a_or_b_on_tail {
    ((a: $a:ident, b: $b:ident), call a: $($tail:tt)*) => {
        $a(stringify!($($tail)*))
    };

    ((a: $a:ident, b: $b:ident), call b: $($tail:tt)*) => {
        $b(stringify!($($tail)*))
    };

    ($ab:tt, $_skip:tt $($tail:tt)*) => {
        call_a_or_b_on_tail!($ab, $($tail)*)
    };
}

fn compute_len(s: &str) -> Option<usize> {
    Some(s.len())
}

fn show_tail(s: &str) -> Option<usize> {
    println!("tail: {:?}", s);
    None
}

fn main() {
    assert_eq!(
        call_a_or_b_on_tail!(
            (a: compute_len, b: show_tail),
            the recursive part that skips over all these
            tokens doesn't much care whether we will call a
            or call b: only the terminal rules care.
        ),
        None
    );
    assert_eq!(
        call_a_or_b_on_tail!(
            (a: compute_len, b: show_tail),
            and now, to justify the existence of two paths
            we will also call a: its input should somehow
            be self-referential, so let's make it return
            some eighty-six!
        ),
        Some(92)
    );
}
```

在十分复杂的递归宏中，可能需要非常多的参数，
才足以在每层调用之间传递必要的标识符与表达式。
然而，根据实现上的差异，可能存在许多这样的中间层，
它们转发了 (forward) 这些参数，但并没有用到。

因此，将所有这些参数捆绑 (bundle) 在一起，通过分组将其放进单独一棵标记树 [`tt`] 里，
可以省事许多。这样一来，那些用不到这些参数的递归层可以直接捕获并替换这棵标记树，
而不需要把整组参数完完全全准准确确地捕获替换掉。

上面的例子把表达式 `$a` 和 `$b` 捆绑起来，
然后作为一棵 [`tt`] 交由递归规则处理。
随后，终结规则 (terminal rules) 将这组标记解构 (destructure) ，
并访问其中的表达式。

[`tt`]: ../minutiae/fragment-specifiers.html#tt
