# Never type
# never类型

>[never.md](https://github.com/rust-lang/reference/blob/master/src/types/never.md)\
>commit 91486df597a9e8060f9c67587359e9f168dea7ef

> **<sup>句法</sup>**\
> _NeverType_ : `!`

never类型(`!`)是一个没有值的类型，表示永远不会完成的计算的计算结果。`!`的类型表达式可以强制转换为任何其他类型。

<!-- ignore: unstable -->
```rust,ignore
let x: ! = panic!();
// 可以强制转换为任何类型
let y: u32 = x;
```

**注意：** never类型本预计在1.41中稳定下来，但由于最后一分钟检测到一些意想不到的回归，该类型的稳定进程临时暂停。目前 `!`类型只能出现在函数返回类型中。有关详细信息，请参阅[问题跟踪](https://github.com/rust-lang/rust/issues/35121)。
