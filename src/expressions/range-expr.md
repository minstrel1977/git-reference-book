# Range expressions
# 区间表达式

>[range-expr.md](https://github.com/rust-lang/reference/blob/master/src/expressions/range-expr.md)\
>commit 1cc592ee270b4d9ad190a8cacce0a1ed356b54d0

> **<sup>句法</sup>**\
> _RangeExpression_ :\
> &nbsp;&nbsp; &nbsp;&nbsp; _RangeExpr_\
> &nbsp;&nbsp; | _RangeFromExpr_\
> &nbsp;&nbsp; | _RangeToExpr_\
> &nbsp;&nbsp; | _RangeFullExpr_\
> &nbsp;&nbsp; | _RangeInclusiveExpr_\
> &nbsp;&nbsp; | _RangeToInclusiveExpr_
>
> _RangeExpr_ :\
> &nbsp;&nbsp; [_Expression_] `..` [_Expression_]
>
> _RangeFromExpr_ :\
> &nbsp;&nbsp; [_Expression_] `..`
>
> _RangeToExpr_ :\
> &nbsp;&nbsp; `..` [_Expression_]
>
> _RangeFullExpr_ :\
> &nbsp;&nbsp; `..`
>
> _RangeInclusiveExpr_ :\
> &nbsp;&nbsp; [_Expression_] `..=` [_Expression_]
>
> _RangeToInclusiveExpr_ :\
> &nbsp;&nbsp; `..=` [_Expression_]

`..`和 `..=`操作符将根据下表构造各种 `std::ops::Range`(或 `core::ops::Range`)的变体的对象：

| 表达式分类              | 句法          | 类型                          | 区间语义              |
|------------------------|---------------|------------------------------|-----------------------|
| _RangeExpr_            | start`..`end  | [std::ops::Range]            | start &le; x &lt; end |
| _RangeFromExpr_        | start`..`     | [std::ops::RangeFrom]        | start &le; x          |
| _RangeToExpr_          | `..`end       | [std::ops::RangeTo]          |            x &lt; end |
| _RangeFullExpr_        | `..`          | [std::ops::RangeFull]        |            -          |
| _RangeInclusiveExpr_   | start`..=`end | [std::ops::RangeInclusive]   | start &le; x &le; end |
| _RangeToInclusiveExpr_ | `..=`end      | [std::ops::RangeToInclusive] |            x &le; end |

举例：

```rust
1..2;   // std::ops::Range
3..;    // std::ops::RangeFrom
..4;    // std::ops::RangeTo
..;     // std::ops::RangeFull
5..=6;  // std::ops::RangeInclusive
..=7;   // std::ops::RangeToInclusive
```

下面的表达式是等价的。

```rust
let x = std::ops::Range {start: 0, end: 10};
let y = 0..10;

assert_eq!(x, y);
```

区间能在 `for`循环里使用：

```rust
for i in 1..11 {
    println!("{}", i);
}
```

[_Expression_]: ../expressions.md

[std::ops::Range]: https://doc.rust-lang.org/std/ops/struct.Range.html
[std::ops::RangeFrom]: https://doc.rust-lang.org/std/ops/struct.RangeFrom.html
[std::ops::RangeTo]: https://doc.rust-lang.org/std/ops/struct.RangeTo.html
[std::ops::RangeFull]: https://doc.rust-lang.org/std/ops/struct.RangeFull.html
[std::ops::RangeInclusive]: https://doc.rust-lang.org/std/ops/struct.RangeInclusive.html
[std::ops::RangeToInclusive]: https://doc.rust-lang.org/std/ops/struct.RangeToInclusive.html
