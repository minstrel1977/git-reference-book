# Dynamically Sized Types
# 动态尺寸类型

>[dynamically-sized-types.md](https://github.com/rust-lang/reference/blob/master/src/dynamically-sized-types.md)\
>commit: 83f725f1b9dda6166589d7b715b75b7f54143b8e \
>本章译文最后维护日期：2021-07-31

大多数的类型都有一个在编译时就已知的固定尺寸，并实现了 trait [`Sized`][sized]。只有在运行时才知道尺寸的类型称为*动态尺寸类型(dynamically sized type)*（*DST*），或者非正式地称为非固定尺寸类型(unsized type)。[切片][Slices]和 [trait对象][trait objects]是 <abbr title="dynamically sized types">DSTs</abbr> 的两个例子。此类类型只能在某些情况下使用:

* 指向 <abbr title="dynamically sized types">DST</abbr> 的[指针类型][Pointer types]的尺寸是固定的(sized)，但是是指向固定尺寸类型的指针的尺寸的两倍
    * 指向切片的指针也存储了切片的元素的数量。
    * 指向 trait对象的指针也存储了一个指向虚函数表(vtable)的指针地址
* <abbr title="dynamically sized types">DST</abbr> 可以作为类型实参( type arguments)来传给有 `?Sized`约束的泛型参数。
    当关联类型的声明有 `？size`约束时，它们也可以被用于关联类型定义。
    默认情况下，任何类型参数或关联类型都有 `Sized`约束，除非它们使用了 `Sized`来放宽其约束。
* 可以为 <abbr title="dynamically sized types">DSTs</abbr> 实现 trait。
    与泛型类型参数中的默认设置不同，在 trait定义中，`Self: ?Sized`约束是默认情况。
* 结构体可以包含一个 <abbr title="dynamically sized type">DST</abbr> 作为最后一个字段；这使得该结构体也成为是一个 <abbr title="dynamically sized type">DST</abbr>。

> **注意**：[变量][variables]、函数参数、[常量][const]项和[静态][static]项必须是 `Sized`。

[sized]: special-types-and-traits.md#sized
[Slices]: types/slice.md
[trait objects]: types/trait-object.md
[Pointer types]: types/pointer.md
[variables]: variables.md
[const]: items/constant-items.md
[static]: items/static-items.md