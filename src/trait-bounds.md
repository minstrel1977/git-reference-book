# Trait and lifetime bounds
# trait约束和生存期约束

>[trait-bounds.md](https://github.com/rust-lang/reference/blob/master/src/trait-bounds.md)\
>commit: f8e76ee9368f498f7f044c719de68c7d95da9972 \
>本章译文最后维护日期：2020-10-31

> **<sup>句法</sup>**\
> _TypeParamBounds_ :\
> &nbsp;&nbsp; _TypeParamBound_ ( `+` _TypeParamBound_ )<sup>\*</sup> `+`<sup>?</sup>
>
> _TypeParamBound_ :\
> &nbsp;&nbsp; &nbsp;&nbsp; _Lifetime_ | _TraitBound_
>
> _TraitBound_ :\
> &nbsp;&nbsp; &nbsp;&nbsp; `?`<sup>?</sup>
> [_ForLifetimes_](#higher-ranked-trait-bounds)<sup>?</sup> [_TypePath_]\
> &nbsp;&nbsp; | `(` `?`<sup>?</sup>
> [_ForLifetimes_](#higher-ranked-trait-bounds)<sup>?</sup> [_TypePath_] `)`
>
> _LifetimeBounds_ :\
> &nbsp;&nbsp; ( _Lifetime_ `+` )<sup>\*</sup> _Lifetime_<sup>?</sup>
>
> _Lifetime_ :\
> &nbsp;&nbsp; &nbsp;&nbsp; [LIFETIME_OR_LABEL]\
> &nbsp;&nbsp; | `'static`\
> &nbsp;&nbsp; | `'_`

[trait][Trait]约束和生存期约束为[泛型项][generic]提供了一种方法来限制将哪些类型和生存期可被用作它们的参数。通过 [where子句][where clause]可以为任何泛型提供约束。对于某些常见的情况，也可以使用如下简写形式：

* 跟在[泛型参数][generic]声明之后的约束：`fn f<A: Copy>() {}` 与 `fn f<A> where A: Copy () {}` 效果一样。
* 在 trait声明中作为指定[超类trait(supertraits)][supertraits] 约束时：`trait Circle : Shape {}` 等同于 `trait Circle where Self : Shape {}`。
* 在 trait声明中作为指定关联类型上的约束时：`trait A { type B: Copy; }` 等同于 `trait A where Self::B: Copy { type B; }`。

在数据项上应用了约束就要求在使用该数据项时使用者必须满足这些约束。当对泛型项进行类型检查和借用检查时，约束可用来确认当前准备用来单态化此泛型的实例类型是否实现了约束给出的 trait。例如，给定 `Ty: Trait`：

* 在泛型函数体中，`Trait` 中的方法可以被 `Ty`类型的值调用。同样，`Trait` 上的相关常数也可以被使用。
* `Trait` 上的关联类型可以被使用。
* 带有 `T: Trait`约束的泛型函数或类型可以在使用 `T` 的地方替换使用 `Ty`。

```rust
# type Surface = i32;
trait Shape {
    fn draw(&self, Surface);
    fn name() -> &'static str;
}

fn draw_twice<T: Shape>(surface: Surface, sh: T) {
    sh.draw(surface);           // 能调用此方法上因为 T: Shape
    sh.draw(surface);
}

fn copy_and_draw_twice<T: Copy>(surface: Surface, sh: T) where T: Shape {
    let shape_copy = sh;        // sh 没有被使用移动语义移走，是因为 T: Copy
    draw_twice(surface, sh);    // 能使用泛型函数 draw_twice 是因为 T: Shape
}

struct Figure<S: Shape>(S, S);

fn name_figure<U: Shape>(
    figure: Figure<U>,          // 这里类型 Figure<U> 的格式正确是因为 U: Shape
) {
    println!(
        "Figure of two {}",
        U::name(),              // 可以使用关联函数
    );
}
```

trait 和生存期约束也被用来命名 [trait对象][trait objects]。

## `?Sized`

`?` 仅用于声明 [`Sized`] trait 可能不会被某类型参数或关联类型实现。`?Sized` 还不能用作其他类型的约束。

## Lifetime bounds
## 生存期约束

生存期约束可以应用于类型或其他生存期。约束 `'a: 'b` 通常被解读为 *`'a` 比 `'b` 存活的时间久*。`'a: 'b`意味着 `'a` 存活的时间比 `'b` 长，所以只要 `&'b ()` 有效，引用 `&'a ()` 就有效。

```rust
fn f<'a, 'b>(x: &'a i32, mut y: &'b i32) where 'a: 'b {
    y = x;                      // 因为 'a: 'b，所以&'a i32 是 &'b i32 的子类型
    let r: &'b &'a i32 = &&0;   // &'b &'a i32 格式合法是因为 'a: 'b
}
```
`T: 'a` 意味着 `T` 的所有生存期参数都比 `'a` 存活得时间长。例如，如果 `'a` 是一个任意的(unconstrained)生存期参数，那么 `i32: 'static` 和 `&'static str: 'a` 都合法，但 `Vec<&'a ()>: 'static` 不合法。

## Higher-ranked trait bounds
## 高阶trait约束

可以在生存期上再进行更高阶的类型约束。这些高阶约束指定了一个对*所有*生存期都为真的约束。例如，像 `for<'a> &'a T: PartialEq<i32>` 这样的约束需要一个如下的实现
<!-- 高阶trait约束是对带生存期的类型进行了进一步约束，像这句里的例子就是对`&'a T`加上了PartialEq<i32>的约束，其中for<'a>可以理解为：对于'a的所有可能选择。https://doc.rust-lang.org/std/cmp/trait.PartialEq.html 和 https://doc.rust-lang.org/nightly/nomicon/hrtb.html -->
```rust
# struct T;
impl<'a> PartialEq<i32> for &'a T {
    // ...
#    fn eq(&self, other: &i32) -> bool {true}
}
```

然后就可以比较任意生存期的 `&'a T` 和 `i32` 啦。

下面这类场景只能使用高阶trait约束，因为引用的生命周期比函数的生命周期参数短：

```rust
fn call_on_ref_zero<F>(f: F) where for<'a> F: Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```
<!-- ```rust compile_error 这里F就没约束'a,导致参数f引用了未经扩展（生存期）的生存期更短的zero
fn call_on_ref_zero<'a, F>(f: F) where  F: Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
``` -->
高阶生存期也可以贴近 trait 来指定，唯一的区别是生存期参数的作用域，像下面这样 `'a` 的作用域只扩展到后面跟的 trait 的末尾，而不是整个约束。下面这个函数和上一个等价。<!-- 这句的意思是如果F的约束有多个trait，那'a的作用域只是扩展它后面紧跟的那个trait的方法 -->

```rust
fn call_on_ref_zero<F>(f: F) where F: for<'a> Fn(&'a i32) {
    let zero = 0;
    f(&zero);
}
```

[LIFETIME_OR_LABEL]: tokens.md#lifetimes-and-loop-labels
[_TypePath_]: paths.md#paths-in-types
[`Sized`]: special-types-and-traits.md#sized

[associated types]: items/associated-items.md#associated-types
[supertraits]: items/traits.md#supertraits
[generic]: items/generics.md
[Trait]: items/traits.md#trait-bounds
[trait objects]: types/trait-object.md
[where clause]: items/generics.md#where-clauses

<!-- 2020-11-7-->
<!-- checked -->
