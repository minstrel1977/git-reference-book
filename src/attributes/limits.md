# Limits
# 极限值设置

>[limits.md](https://github.com/rust-lang/reference/blob/master/src/attributes/limits.md)\
>commit: a258c97b95c9c1bf27f4b6684e5b5d080eb8aa69 \
>本译文最后维护日期：2020-10-24

以下[属性][attributes]影响编译期的编译参数的极限值设置。

## The `recursion_limit` attribute
## `recursion_limit`属性

*`recursion_limit`属性*可以应用于 [crate] 级别，为可能无限递归的编译期操作（如宏扩展或自动解引用）设置最大递归深度。它使用 [_MetaNameValueStr_]元项属性句法规则来指定递归深度。

> 注意：`rustc` 中的这个默认值是128。

```rust,compile_fail
#![recursion_limit = "4"]

macro_rules! a {
    () => { a!(1) };
    (1) => { a!(2) };
    (2) => { a!(3) };
    (3) => { a!(4) };
    (4) => { };
}

// 这无法扩展，因为它需要大于4的递归深度。
a!{}
```

```rust,compile_fail
#![recursion_limit = "1"]

// 这里的失败是因为需要两个递归步骤来自动解引用
(|_: &u8| {})(&&1);
```

## The `type_length_limit` attribute
## `type_length_limit`属性

*`type_length_limit`属性*限制在单态化过程中构造具体类型时所做的最大类型替换次数。它应用于 [crate] 级别，并使用 [_MetaNameValueStr_]元项属性句法规则来设置类型替换数量的上限。

> 注意：`rustc` 中的这个默认值是1048576。

<!-- This code should fail to compile. Unfortunately rustdoc's `compile_fail` stops after analysis phase, and this error is generated after that. So this needs to be `ignore` for now. -->

```rust,compile_fail,ignore
#![type_length_limit = "8"]

fn f<T>(x: T) {}

// 这里的编译失败是因为单态化 `f::<(i32, i32, i32, i32, i32, i32, i32, i32, i32)>>` 需要大于8个类型元素。
f((1, 2, 3, 4, 5, 6, 7, 8, 9));
```

[_MetaNameValueStr_]: ../attributes.md#meta-item-attribute-syntax
[attributes]: ../attributes.md
[crate]: ../crates-and-source-files.md

<!-- 2020-10-25 -->
<!-- checked -->
