# Expressions
# 表达式

>[expressions.md](https://github.com/rust-lang/reference/blob/master/src/expressions.md)\
>commit: 60ccf42d69d4378437afa9d35b6b95394649735c \
>本章译文最后维护日期：2024-10-13

> **<sup>句法</sup>**\
> _Expression_ :\
> &nbsp;&nbsp; &nbsp;&nbsp; _ExpressionWithoutBlock_\
> &nbsp;&nbsp; | _ExpressionWithBlock_
>
> _ExpressionWithoutBlock_ :\
> &nbsp;&nbsp; [_OuterAttribute_]<sup>\*</sup>[†](#expression-attributes)\
> &nbsp;&nbsp; (\
> &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; [_LiteralExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_PathExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_OperatorExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_GroupedExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_ArrayExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_AwaitExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_IndexExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_TupleExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_TupleIndexingExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_StructExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_CallExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_MethodCallExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_FieldExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_ClosureExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_AsyncBlockExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_ContinueExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_BreakExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_RangeExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_ReturnExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_UnderscoreExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_MacroInvocation_]\
> &nbsp;&nbsp; )
>
> _ExpressionWithBlock_ :\
> &nbsp;&nbsp; [_OuterAttribute_]<sup>\*</sup>[†](#expression-attributes)\
> &nbsp;&nbsp; (\
> &nbsp;&nbsp; &nbsp;&nbsp; &nbsp;&nbsp; [_BlockExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_ConstBlockExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_UnsafeBlockExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_LoopExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_IfExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_IfLetExpression_]\
> &nbsp;&nbsp; &nbsp;&nbsp; | [_MatchExpression_]\
> &nbsp;&nbsp; )

一个表达式可以扮演两个角色：它总是能产生一个*值*；它还有可能表达出*效果(effects)*（也被称为“副作用(side effects)”）。
表达式会被 _求值/计算为(evaluates to)_ 为值，并在 _求值(evaluation)_ 期间表达出效果。 
很多表达式都会包含子表达式，此时子表达式也被称为此表达式的*操作数*。
每种表达式都表达了以下几点含义：

* 在对表达式求值时是否对操作数求值
* 对操作数求值的顺序
* 如何组合操作数的值来获取表达式的值

基于对这几种含义的实现要求，表达式通过其内在结构规定了其执行结构。
块只是另一种表达式，所以块、语句和表达式可以递归地彼此嵌套到任意深度。

> **注意**：我们给表达式的操作数做了（如上）命名，以便于我们去讨论它们，但这些名称并没有最终稳定下来，以后可能还会更改。

## Expression precedence
## 表达式的优先级

Rust 运算符和表达式的优先级顺序如下，从强到弱。
具有相同优先级的二元运算符按其结合(associativity)顺序做了分组。

| 运算符/表达式         | 结合性       |
|-----------------------------|---------------------|
| Paths（路径）                |                     |
| Method calls（方法调用）     |                     |
| Field expressions （字段表达式）| 从左向右       |
| Function calls, array indexing（函数调用，数组索引）|                  |
| `?`                         |                     |
| Unary（一元运算符） `-` `*` `!` `&` `&mut` |                    |
| `as`                        | 从左向右       |
| `*` `/` `%`                 | 从左向右       |
| `+` `-`                     | 从左向右       |
| `<<` `>>`                   | 从左向右       |
| `&`                         | 从左向右       |
| `^`                         | 从左向右       |
| <code>&#124;</code>         | 从左向右       |
| `==` `!=` `<` `>` `<=` `>=` | 需要圆括号 |
| `&&`                        | 从左向右       |
| <code>&#124;&#124;</code>   | 从左向右       |
| `..` `..=`                  | 需要圆括号 |
| `=` `+=` `-=` `*=` `/=` `%=` <br> `&=` <code>&#124;=</code> `^=` `<<=` `>>=` | 从右向左 |
| `return` `break` closures（返回、中断、闭包）   |                     |

## Evaluation order of operands
## 操作数的求值顺序

下面的表达式列表都以相同的方式计算它们的操作数，具体列表后面也有详述。
其他表达式要么不接受操作数，要么按照各自约定（后续章节会有讲述）的条件进行求值。

* 解引用表达式(Dereference expression)
* 错误传播表达式(Error propagation expression)
* 取反表达式(Negation expression)
* 算术和二进制逻辑运算(Arithmetic and logical binary operators)
* 比较运算(Comparison operators)
* 类型转换表达式(Type cast expression)
* 分组表达式(Grouped expression)
* 数组表达式(Array expression)
* 等待表达式(Await expression)
* 索引表达式(Index expression)
* 元组表达式(Tuple expression)
* 元组索引表达式(Tuple index expression)
* 结构体表达式(Struct expression)
* 调用表达式(Call expression)
* 方法调用表达式(Method call expression)
* 字段表达式(Field expression)
* 中断表达式(Break expression)
* 区间表达式(Range expression)
* 返回表达式(Return expression)

在实现执行这些表达式的效果之前，会先对这些表达式的操作数进行求值。
拥有多个操作数的表达式会按照源代码书写的顺序从左到右计算。

> **注意**：子表达式是一个表达式的操作数时，此子表达式内部的求值顺序是由根据前面的章节规定的优先级来确定的。

例如，下面两个 `next`方法的调用总是以相同的顺序调用：

```rust
# // 使用 vec 代替 array 来避免引用，因为在编写本例时，拥有内部元素的所有权的数组迭代器还没有稳定下来。
let mut one_two = vec![1, 2].into_iter();
assert_eq!(
    (1, 2),
    (one_two.next().unwrap(), one_two.next().unwrap())
);
```

> **注意**：由于表达式是递归执行的，那这些表达式也会从最内层到最外层逐层求值，忽略兄弟表达式，直到没有（未求值的）内部子表达式为止。

## Place Expressions and Value Expressions
## 位置表达式和值表达式

表达式分为两大类：位置表达式和值表达式。跟在每个表达式中一样，操作数可以出现在位置上下文，也可出现在值上下文中。
表达式的求值既依赖于它自己的类别，也依赖于它所在的上下文。
表达式分为两大类别：位置表达式和值表达式；还有第三个小类表达式，称为 assignee表达式。
在每类表达式中，操作数可以出现在位置上下文中，也可以出现在值上下文中。
表达式的求值既取决于其自身的类别，也取决于其处于的上下文。

*位置表达式*是表示内存位置的表达式。语言中表现为指向**局部变量、[静态变量][static variables]、[解引用][deref](`*expr`)、[数组索引][array indexing]表达式(`expr[expr]`)、[字段][field]引用(`expr.f`)、圆括号括起来的位置表达式**的[路径][paths]。
所有其他形式的表达式都是值表达式。

*值表达式*是代表实际值的表达式。

下面的上下文是*位置表达式*上下文：

* [复合赋值][compound assignment]表达式的左操作数。
* 一元运算符[借用][borrow]、[裸借用][raw borrow]或[解引用][deref]的操作数。
* 字段表达式的操作数。
* 数组索引表达式的被索引操作数。
* 任何[隐式借用][implicit borrow]的操作数。
* [let语句][let statement]的初始化器(initializer)。
* [`if let`]表达式、[`while let`]表达式或[匹配(`match`)][match]表达式的[检验对象(scrutinee)][scrutinee]。
* 结构体表达式里的[函数式更新(functional update)][functional update]的基(base)。

> 注意：历史上，位置表达式被称为 *左值/lvalues*，值表达式被称为 *右值/rvalues*。

*assignee表达式*是出现在[赋值][assign]表达式的左操作数中的表达式。确切地说，assignee表达式是：

- 位置表达式。
- [下划线][_UnderscoreExpression_].
- 出现在 assignee表达式里的[元组][_TupleExpression_]。
- 出现在 assignee表达式里的[切片][_ArrayExpression_]。
- 出现在 assignee表达式里的[元组结构体][_StructExpression_] 。
- 出现在 assignee表达式（字段名可选）里的[结构体][_StructExpression_] 。
- [单元结构体][_StructExpression_].

assignee表达式允许使用任意层次的圆括号嵌套。

### Moved and copied types
### 移动语义类型和复制语义类型

当位置表达式在值表达式上下文中被求值时，或在模式中被值绑定时，这表示此值会*保存进(held in)*当前表达式代表的内存地址。
如果该值的类型实现了 [`Copy`]，那么该值将被从原来的位置复制一份过来。
如果该值的类型没有实现 [`Copy`]，但实现了 [`Sized`]，那么就有可能把该值从原来的位置移动(move)过来。
从位置表达式里移出值对位置表达式也有要求，只有如下的位置表达式才可能把值从其中移出(move out)：

* 当前未被借用的[变量][Variables]。
* [临时变量里的值(Temporary values)](#temporaries)。
* 表达式本身就可以被移出值，那它的一些字段也可被移出，但要求这些字段（的类型）没实现 [`Drop`]。
* 对可移出且类型为 [`Box<T>`] 的表达式作[解引用][deref]的结果。

当把值从一个位置表达式里移出到一个局部变量中后，此（表达式代表的）地址将被去初始化(deinitialized)，并且该地址在重新初始化之前无法被再次读取。
除以上列出的情况外，任何在值表达式上下文中使用位置表达式都是错误的。

### Mutability
### 可变性

如果一个位置表达式将会被[赋值][assign]、可变[借出][borrow]、[隐式可变借出][implicitly mutably borrowed]或被绑定到包含 `ref mut` 模式上，则该位置表达式必须是*可变的(mutable)*。我们称这类位置表达式为*可变位置表达式*。
与之相对，其他位置表达式称为*不可变位置表达式*。

下面的表达式可以是可变位置表达式上下文：

* 当前未被借出的可变[变量][variables]。
* [可变静态(`static`)项][Mutable `static` items]。
* [临时值][Temporary values]。
* [字段][field]：在可变位置表达式上下文中，会对子表达式求值。
* 对 `*mut T` 指针的[解引用][deref]。
* 对类型为 `&mut T` 的变量或变量的字段的解引用。注意：这条是下一条规则的必要条件的例外情况。
* 对实现 `DerefMut` 的类型的解引用：这就要求在可变位置表达式上下文中来处理这个被解出的值。
* 对实现 `IndexMut` 的类型做[索引][array indexing]，那对此检索出的表达式求值就需要在一个可变位置表达式上下文进行。注意对索引(index)本身的求值不用。

### Temporaries
### 临时位置/临时变量

在大多数位置表达式上下文中使用值表达式时，会创建一个临时的未命名内存位置，并将该位置初始为该值，然后这个表达式的求值结果就为该内存位置。
此过程也有例外，就是把此临时表达式[提升][promoted]为一个静态项(`static`)。（译者注：这种情况下表达式将直接在编译时就求值了，求值的结果会根据编译器要求重新选择地址存储）。
临时位置/临时变量的[存续作用域(drop scope)][drop scope]通常在包含它的最内层语句的结尾处。

### Implicit Borrows
### 隐式借用

某些特定的表达式可以通过隐式借用一个表达式来将其视为位置表达式。例如，可以直接比较两个非固定内存宽度的[切片][slice]是否相等，因为 `==` 操作符隐式借用了它自身的操作数：

```rust
# let c = [1, 2, 3];
# let d = vec![1, 2, 3];
let a: &[i32];
let b: &[i32];
# a = &c;
# b = &d;
// ...
*a == *b; //译者注：&[i32] 解引用后是一个动态内存宽度类型，理论上两个动态内存宽度类型上无法比较大小的，但这里因为隐式借用此成为可能
// 等价于下面的形式:
::std::cmp::PartialEq::eq(&*a, &*b);
```

隐式借用可能会被以下表达式采用：

* [方法调用][method-call]表达式中的左操作数。
* [字段][field]表达式中的左操作数。
* [调用表达式][call expressions]中的左操作数。
* [数组索引][array indexing]表达式中的左操作数。
* [解引用操作符][deref]（`*`）的操作数。
* [比较运算][comparison]的操作数。
* [复合赋值(compound assignment)][compound assignment]的左操作数。

## Overloading Traits
## 重载trait

本书后续章节的许多操作符和表达式都可以通过使用 `std::ops` 或 `std::cmp` 中的 trait 被其他类型重载。
这些 trait 也存在于同名的 `core::ops` 和 `core::cmp` 中。

## Expression Attributes
## 表达式属性

只有在少数特定情况下，才允许在表达式之前使用[外部属性][_OuterAttribute_]：

* 在被当作[语句][statement]用的表达式之前。
* [数组表达式][array expressions]、[元组表达式][tuple expressions]、[调用表达式][call expressions]、[元组结构体(tuple-style struct)][struct]表达式这些中的元素。
* [块表达式][block expressions]的尾部表达式(tail expression)。
<!-- 本列表需要和 block-expr.md 保持同步-->

在下面情形之前是不允许的：
* [区间(Range)][_RangeExpression_]表达式。
* 二元运算符表达式（[_ArithmeticOrLogicalExpression_]、[_ComparisonExpression_]、[_LazyBooleanExpression_]、[_TypeCastExpression_]、[_AssignmentExpression_]、[_CompoundAssignmentExpression_]）。

[block expressions]:    expressions/block-expr.md
[call expressions]:     expressions/call-expr.md
[field]:                expressions/field-expr.md
[functional update]:    expressions/struct-expr.md#functional-update-syntax
[`if let`]:             expressions/if-expr.md#if-let-expressions
[match]:                expressions/match-expr.md
[method-call]:          expressions/method-call-expr.md
[paths]:                expressions/path-expr.md
[struct]:               expressions/struct-expr.md
[tuple expressions]:    expressions/tuple-expr.md
[`while let`]:          expressions/loop-expr.md#predicate-pattern-loops

[array expressions]:    expressions/array-expr.md
[array indexing]:       expressions/array-expr.md#array-and-slice-indexing-expressions

[assign]:               expressions/operator-expr.md#assignment-expressions
[borrow]:               expressions/operator-expr.md#borrow-operators
[comparison]:           expressions/operator-expr.md#comparison-operators
[compound assignment]:  expressions/operator-expr.md#compound-assignment-expressions
[deref]:                expressions/operator-expr.md#the-dereference-operator

[destructors]:          destructors.md
[drop scope]:           destructors.md#drop-scopes

[`Copy`]:               special-types-and-traits.md#copy
[`Drop`]:               special-types-and-traits.md#drop
[`Sized`]:              special-types-and-traits.md#sized
[implicit borrow]:      #implicit-borrows
[implicitly mutably borrowed]: #implicit-borrows
[interior mutability]:  interior-mutability.md
[let statement]:        statements.md#let-statements
[Mutable `static` items]: items/static-items.md#mutable-statics
[scrutinee]:            glossary.md#scrutinee
[promoted]:             destructors.md#constant-promotion
[raw borrow]:           expressions/operator-expr.md#raw-borrow-operators
[slice]:                types/slice.md
[statement]:            statements.md
[static variables]:     items/static-items.md
[Temporary values]:     #temporaries
[Variables]:            variables.md

[_ArithmeticOrLogicalExpression_]: expressions/operator-expr.md#arithmetic-and-logical-binary-operators
[_ArrayExpression_]:              expressions/array-expr.md
[_AsyncBlockExpression_]:         expressions/block-expr.md#async-blocks
[_AwaitExpression_]:              expressions/await-expr.md
[_AssignmentExpression_]:         expressions/operator-expr.md#assignment-expressions
[_BlockExpression_]:              expressions/block-expr.md
[_BreakExpression_]:              expressions/loop-expr.md#break-expressions
[_CallExpression_]:               expressions/call-expr.md
[_ClosureExpression_]:            expressions/closure-expr.md
[_ComparisonExpression_]:         expressions/operator-expr.md#comparison-operators
[_CompoundAssignmentExpression_]: expressions/operator-expr.md#compound-assignment-expressions
[_ConstBlockExpression_]:         expressions/block-expr.md#const-blocks
[_ContinueExpression_]:           expressions/loop-expr.md#continue-expressions
[_FieldExpression_]:              expressions/field-expr.md
[_GroupedExpression_]:            expressions/grouped-expr.md
[_IfExpression_]:                 expressions/if-expr.md#if-expressions
[_IfLetExpression_]:              expressions/if-expr.md#if-let-expressions
[_IndexExpression_]:              expressions/array-expr.md#array-and-slice-indexing-expressions
[_LazyBooleanExpression_]:        expressions/operator-expr.md#lazy-boolean-operators
[_LiteralExpression_]:            expressions/literal-expr.md
[_LoopExpression_]:               expressions/loop-expr.md
[_MacroInvocation_]:              macros.md#macro-invocation
[_MatchExpression_]:              expressions/match-expr.md
[_MethodCallExpression_]:         expressions/method-call-expr.md
[_OperatorExpression_]:           expressions/operator-expr.md
[_OuterAttribute_]:               attributes.md
[_PathExpression_]:               expressions/path-expr.md
[_RangeExpression_]:              expressions/range-expr.md
[_ReturnExpression_]:             expressions/return-expr.md
[_StructExpression_]:             expressions/struct-expr.md
[_TupleExpression_]:              expressions/tuple-expr.md
[_TupleIndexingExpression_]:      expressions/tuple-expr.md#tuple-indexing-expressions
[_TypeCastExpression_]:           expressions/operator-expr.md#type-cast-expressions
[_UnderscoreExpression_]:         expressions/underscore-expr.md
[_UnsafeBlockExpression_]:        expressions/block-expr.md#unsafe-blocks
