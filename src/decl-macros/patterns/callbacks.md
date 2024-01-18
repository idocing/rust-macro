# 回调

```rust,editable
macro_rules! call_with_larch {
    ($callback:ident) => { $callback!(larch) };
}

macro_rules! expand_to_larch {
    () => { larch };
}

macro_rules! recognize_tree {
    (larch) => { println!("#1, the Larch.") };
    (redwood) => { println!("#2, the Mighty Redwood.") };
    (fir) => { println!("#3, the Fir.") };
    (chestnut) => { println!("#4, the Horse Chestnut.") };
    (pine) => { println!("#5, the Scots Pine.") };
    ($($other:tt)*) => { println!("I don't know; some kind of birch maybe?") };
}

fn main() {
    recognize_tree!(expand_to_larch!()); // 无法直接使用 `expand_to_larch!` 的展开结果
    call_with_larch!(recognize_tree);    // 回调就是给另一个宏传入宏的名称 (`ident`)，而不是宏的结果
}

// 打印结果：
// I don't know; some kind of birch maybe?
// #1, the Larch.
```

由于宏展开的机制限制，（至少在最新的 Rust 中）
不可能做到把一例宏的展开结果作为有效信息提供给另一例宏。
这为宏的模块化工作施加了难度。

使用递归并传递回调 (callbacks) 是条出路。
作为演示，上例两处宏调用的展开过程如下：

```rust,ignore
recognize_tree! { expand_to_larch ! (  ) }
println! { "I don't know; some kind of birch maybe?" }
// ...

call_with_larch! { recognize_tree }
recognize_tree! { larch }
println! { "#1, the Larch." }
// ...
```

可以反复匹配 `tt` 来将任意参数转发给回调：

```rust,editable
macro_rules! callback {
    ($callback:ident( $($args:tt)* )) => {
        $callback!( $($args)* )
    };
}

fn main() {
    callback!(callback(println("Yes, this *was* unnecessary.")));
}
```

如果需要的话，当然还可以在参数中增加额外的标记 (tokens) 。
