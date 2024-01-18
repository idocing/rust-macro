# 下推累积

```rust
macro_rules! init_array {
    (@accum (0, $_e:expr) -> ($($body:tt)*))
        => {init_array!(@as_expr [$($body)*])};
    (@accum (1, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (0, $e) -> ($($body)* $e,))};
    (@accum (2, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (1, $e) -> ($($body)* $e,))};
    (@accum (3, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (2, $e) -> ($($body)* $e,))};
    (@as_expr $e:expr) => {$e};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            init_array!(@accum ($n, e.clone()) -> ())
        }
    };
}

let strings: [String; 3] = init_array![String::from("hi!"); 3];
# assert_eq!(format!("{:?}", strings), "[\"hi!\", \"hi!\", \"hi!\"]");
```

在 Rust 中，所有宏最终 **必须** 展开为一个完整、有效的句法元素（比如表达式、条目等等）。
这意味着，不可能定义一个最终展开为残缺构造的宏。

有些人可能希望，上例中的宏能被更加直截了当地表述成：

```ignore
macro_rules! init_array {
    (@accum 0, $_e:expr) => {/* empty */};
    (@accum 1, $e:expr) => {$e};
    (@accum 2, $e:expr) => {$e, init_array!(@accum 1, $e)};
    (@accum 3, $e:expr) => {$e, init_array!(@accum 2, $e)};
    [$e:expr; $n:tt] => {
        {
            let e = $e;
            [init_array!(@accum $n, e)]
        }
    };
}
```

他们预期的展开过程如下：

```rust,ignore
            [init_array!(@accum 3, e)]
            [e, init_array!(@accum 2, e)]
            [e, e, init_array!(@accum 1, e)]
            [e, e, e]
```

然而，这一思路中，每个中间步骤的展开结果都是一个不完整的表达式。
即便这些中间结果对外部来说绝不可见，Rust 仍然禁止这种用法。

下推累积 (push-down accumulation) 则使我们得以在完全完成之前毋需考虑构造的完整性，
进而累积构建出我们所需的标记序列。
本章开头给出的示例中，宏调用的展开过程如下：

```rust,ignore
init_array! { String:: from ( "hi!" ) ; 3 }
init_array! { @ accum ( 3 , e . clone (  ) ) -> (  ) }
init_array! { @ accum ( 2 , e.clone() ) -> ( e.clone() , ) }
init_array! { @ accum ( 1 , e.clone() ) -> ( e.clone() , e.clone() , ) }
init_array! { @ accum ( 0 , e.clone() ) -> ( e.clone() , e.clone() , e.clone() , ) }
init_array! { @ as_expr [ e.clone() , e.clone() , e.clone() , ] }
```

可以修改一下代码，看到每次调用时 `$($body)*` 存储的内容变化：

```rust,editable
macro_rules! init_array {
    (@accum (0, $_e:expr) -> ($($body:tt)*))
        => {init_array!(@as_expr [$($body)*])};
    (@accum (1, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (0, $e) -> ($($body)* $e+3,))};
    (@accum (2, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (1, $e) -> ($($body)* $e+2,))};
    (@accum (3, $e:expr) -> ($($body:tt)*))
        => {init_array!(@accum (2, $e) -> ($($body)* $e+1,))};
    (@as_expr $e:expr) => {$e};
    [$e:expr; $n:tt $(; first $init:expr)?] => {
        {
            let e = $e;
            init_array!(@accum ($n, e.clone()) -> ($($init)?,))
        }
    };
}

fn main() {
    let array: [usize; 4] = init_array![0; 3; first 0];
    println!("{:?}", array);
}
```

根据 [调试](../macros/minutiae/debugging.md) 一章的内容，
在 nightly Rust 中使用编译命令：
`cargo rustc --bin my-project -- -Z trace-macros` ，即得到以下输出：

```rust,ignore
note: trace_macro
  --> src/main.rs:20:31
   |
20 |     let array: [usize; 4] = init_array![0; 3; first 0];
   |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: expanding `init_array! { 0 ; 3 ; first 0 }`
   = note: to `{ let e = 0 ; init_array! (@ accum(3, e.clone()) -> (0,)) }`
   = note: expanding `init_array! { @ accum(3, e.clone()) -> (0,) }`
   = note: to `init_array! (@ accum(2, e.clone()) -> (0, e.clone() + 1,))`
   = note: expanding `init_array! { @ accum(2, e.clone()) -> (0, e.clone() + 1,) }`
   = note: to `init_array! (@ accum(1, e.clone()) -> (0, e.clone() + 1, e.clone() + 2,))`
   = note: expanding `init_array! { @ accum(1, e.clone()) -> (0, e.clone() + 1, e.clone() + 2,) }`
   = note: to `init_array!
           (@ accum(0, e.clone()) -> (0, e.clone() + 1, e.clone() + 2, e.clone() + 3,))`
   = note: expanding `init_array! { @ accum(0, e.clone()) -> (0, e.clone() + 1, e.clone() + 2, e.clone() + 3,) }`
   = note: to `init_array! (@ as_expr [0, e.clone() + 1, e.clone() + 2, e.clone() + 3,])`
   = note: expanding `init_array! { @ as_expr [0, e.clone() + 1, e.clone() + 2, e.clone() + 3,] }`
   = note: to `[0, e.clone() + 1, e.clone() + 2, e.clone() + 3]`
```

可以看到，每一步都在累积输出，直到规则完成，给出完整的表达式。

上述过程的关键点在于，使用 `$($body:tt)*` 来保存输出中间值，
而不触发其它解析机制。采用 `($input) -> ($output)` 
的形式仅是出于传统，用以明示此类宏的作用。

由于可以存储任意复杂的中间结果，
下推累积在构建 [`TT` 撕咬机](./tt-muncher.md) 的过程中经常被用到。
当构造类似于这个例子的宏时，也会结合 [内用规则](./internal-rules.md)。

# 性能建议

下推累积本质上是二次复杂度的。考虑一个包含 100 
个标记树的累加器[^accumulator]，每次调用一个标记树：

* 初始调用将匹配空的累加器
* 调用第一个递归将匹配 1 个标记树累加器
* 调用下一个递归将匹配 2 个标记树累加器
* 以此类推，最多 100 个

这是一个典型的二次复杂度模式，长输入会导致宏延长编译时间。

此外，TT 撕咬机对其输入也是天生的二次复杂度，所以同时使用
TT 撕咬机和下推累积的宏将是双倍二次的！

所有关于 TT 撕咬机的[性能建议](./tt-muncher.md#performance)都适用于下推积累。

一般来说，避免过多地使用它们，并尽可能地让它们的简单。

最后，确保将累加器放在规则的末尾，而不是开头。

这样，如果匹配规则失败，编译器就不必匹配（可能很长的）累加器，从而避免遇到规则中不匹配的部分。这可能会对编译时间产生很大影响。

[^accumulator]: 译者注：accumulator，即使用下推累积方式编写的声明宏。
