# Diagnostic attributes
# 诊断属性

>[diagnostics.md](https://github.com/rust-lang/reference/blob/master/src/attributes/diagnostics.md)\
>commit: 52e0ff3c11260fb86f19e564684c86560eab4ff9 \
>本章译文最后维护日期：2024-10-13

以下[属性][attributes]用于在编译期间控制或生成诊断消息。

## Lint check attributes
## lint检查类属性

（译者注：lint在原文里有时当名词用，有时当动词用，本文统一翻译成名词，意思就是一种被命名的 lint检查模式）

lint检查(lint check)系统命名了一些潜在的不良编码模式，（这些被命名的 lint检查就是一个一个的lint，）例如编写了不可能执行到的代码，就被命名为 unreachable-code lint，编写未提供文档的代码就被命名为 missing_docs lint。`allow`、`expect`、`warn`、`deny` 和 `forbid` 这些能调整代码检查级别的属性被称为 lint级别属性，它们使用 [_MetaListPaths_]元项属性句法来指定接受此级别的各种 lint。代码实体应用了这些带了具体 lint 列表的 lint级别属性，编译器或相关代码检查工具就可以结合这两层属性对这段代码执行相应的代码检查和检查报告。

对任何名为 `C` 的 lint 来说：

* `#[allow(C)]` 会压制对 `C` 的检查，那这样的违规行为就不会被报告，
* `#[expect(C)]` 表示预期会发射 lint `C`。如果此预期没有发生，该属性将抑制 `C` 的发射或发出警告。
* `#[warn(C)]` 警告违反 `C` 的，但继续编译。
* `#[deny(C)]` 遇到违反 `C` 的情况会触发编译器报错(signals an error)，
* `#[forbid(C)]` 与 `deny(C)` 相同，但同时会禁止以后再更改 lint级别，
* 
> 注意：可以通过 `rustc -W help` 找到所有 `rustc` 支持的 lint，以及它们的默认设置，也可以在 [rustc book] 中找到相关文档。

```rust
pub mod m1 {
    // 这里忽略未提供文档的编码行为
    #[allow(missing_docs)]
    pub fn undocumented_one() -> i32 { 1 }

    // 未提供文档的编码行为在这里将触发编译器警告
    #[warn(missing_docs)]
    pub fn undocumented_too() -> i32 { 2 }

    // 未提供文档的编码行为在这里将触发编译器报错
    #[deny(missing_docs)]
    pub fn undocumented_end() -> i32 { 3 }
}
```
Lint属性可以覆盖上一个属性指定的级别，但该级别不能更改禁止性的 Lint。这里的上一个属性是指语法树中更高级别的属性，或者是同一实体上的前一个属性，按从左到右的源代码顺序列出。

此示例展示了如何使用 `allow` 和 `warn` 来打开和关闭一个特定的 lint检查：

```rust
#[warn(missing_docs)]
pub mod m2 {
    #[allow(missing_docs)]
    pub mod nested {
        // 这里忽略未提供文档的编码行为
        pub fn undocumented_one() -> i32 { 1 }

        // 尽管上面允许，但未提供文档的编码行为在这里将触发编译器警告
        #[warn(missing_docs)]
        pub fn undocumented_two() -> i32 { 2 }
    }

    // 未提供文档的编码行为在这里将触发编译器警告
    pub fn undocumented_too() -> i32 { 3 }
}
```

此示例展示了如何使用 `forbid` 或 `expect` 来禁止对该 lint 使用 `allow`：

```rust,compile_fail
#[forbid(missing_docs)]
pub mod m3 {
    // 试图切换到 allow级别将触发编译器报错
    #[allow(missing_docs)]
    /// 返回 2。
    pub fn undocumented_too() -> i32 { 2 }
}
```

> 注意：`rustc`允许在[命令行][rustc-lint-cli]上设置 lint级别，也支持在[setting caps][rustc-lint-caps]中设置 lint报告中的caps。

### Lint Reasons
### lint 的 reason参数

所有的 lint属性都支持一个附加的 `reason`参数，以给出上下文中添加某特定属性的原因。如果以给定的级别发射了此 lint，则 reason 的内容将显示为 lint 消息的一部分。

```rust,edition2015,compile_fail
// 默认情况下允许使用 `keyword_idents`。在这里，我们拒绝它，以避免在更新版本时迁移标识符。
#![deny(
    keyword_idents,
    reason = "we want to avoid these idents to be future compatible"
)]

// Rust2015版允许使用该名称。为了与未来兼容，我们仍然应尽力于避免这种情况。并且这样做还能避免给最终用户带来困惑。
fn dyn() {}
```

下面是另一个示例，其中允许使用 lint，并给出了原因：

```rust
use std::path::PathBuf;

pub fn get_path() -> PathBuf {
    // `allow`属性上的 `reason`参数扮演了给读者提供说明文档的作用
    #[allow(unused_mut, reason = "this is only modified on some platforms")]
    let mut file_name = PathBuf::from("git");

    #[cfg(target_os = "windows")]
    file_name.set_extension("exe");

    file_name
}
```

### The `#[expect]` attribute
### `#[expect]`属性

`#[expect(C)]`属性为 lint `C` 创建了一个预期lint。如果此预期的发生条件真的被满足，那如果在此相同位置的放置一个 `#[warn(C)]`属性，则会触发此`warn(C)`。如果此预期的发生条件未被满足，因为不会发射 lint `C`，则将在此属性处发射 `unfullilled_lint_expectations` lint。

```rust
fn main() {
    // 该`#[expect]`属性创建了一个预期lint，即下面的语句（在lint检查时）会发射 `unused_variables` lint。但此预期lint 的发生条件没被满足， 因为 宏`println!` 使用了 `question`变量。
    // 因此，将在此属性处发射 `unfulfilled_lint_expectations` lint。
    #[expect(unused_variables)]
    let question = "who lives in a pineapple under the sea?";
    println!("{question}");

    // 因为后面的代码从未使用`answer`变量，所以这个 `#[expect]`属性创建了一个发生条件将被满足的预期lint。那在一般情况下会被发射的 `unused_variables` lint 在此却会被抑制住。也就是 lint检查不会对该语句或属性发射告警。
    #[expect(unused_variables)]
    let answer = "SpongeBob SquarePants!";
}
```

预期lint 仅在 lint发射机制抑制了 `expect`属性限定的lint 被发射出去后才算被满足。如果在作用域内使用其他 lint级别属性（如 `allow` 或 `warn`），则 lint发射机制将先相应地处理一遍这些lint，那前面预置的预期lint 的发生条件就不会被满足。
```rust
#[expect(unused_variables)]
fn select_song() {
    // 这将在 `warn`属性定义的 warn级别上发射 `unused_variables`lint。因此此函数上的预期lint 将不会被满足。
    #[warn(unused_variables)]
    let song_name = "Crab Rave";

    // `allow`属性已经抑制了 `unused_variables`lint 的发射。所以这里是 `allow`属性被满足，而函数上的预期lint 未被满足。
    #[allow(unused_variables)]
    let song_creator = "Noisestorm";

    // 此处的 `expect`属性将抑制此变量的 `unused_variables` lint 的发射。所以函数上的预期lint 未被满足，因为本地的预期lint 的发射已经抑制了 `unused_variables`lint 的发射。
    #[expect(unused_variables)]
    let song_version = "Monstercat Release";
}
```

如果 `expect`属性包含多个 lint，则每个 lint 都要单独计算。对于 lint组，如果组中的一个 lint 被发射，就算此期望lint 被满足：

```rust
// 该预期lint 将因为函数内有未被使用的变量而被执行，因为函数内发射的 `unused_variables` lint 属于 `unused` lint组。
#[expect(unused)]
pub fn thoughts() {
    let unused = "I'm running out of examples";
}

pub fn another_example() {
    // 该属性创建两个预期lint。其中`unused_mut`lint 的发射会被如预期那般被抑制，所一个第一个预期lint 被执行。但同时由于后续代码使用了变量`link`，因此 lint检查机制不会发射 `unused_variables` lint 。因此，后面这个预期lint 将得不到满足，并最终将报告告警信息。
    #[expect(unused_mut, unused_variables)]
    let mut link = "https://www.rust-lang.org/";

    println!("Welcome to our community: {link}");
}
```

> 注意：`#[expect(unfulfilled_lint_expectations)]` 的行为当前定义为始终生成 `unfulfilled_lint_expectations` lint。


### Lint groups
### lint组

lint可以被组织成不同命名的组，以便相关lint的等级可以一起调整。使用命名组相当于给出该组中的 lint。

```rust,compile_fail
// 这允许 "unused"组下的所有lint。
#[allow(unused)]
// 这个禁止动作将覆盖前面 "unused"组中的 "unused_must_use" lint。
#[deny(unused_must_use)]
fn example() {
    // 这里不会生成警告，因为 "unused_variables" lint 在 "unused"组下
    let x = 1;
    // 这里会产生报错信息，因为 "unused_must_use" lint 被标记为"deny"，而这行的返回结果未被使用
    std::fs::remove_file("some_file"); // ERROR: unused `Result` that must be used
}
```

有一个名为 “warnings” 的特殊组，它包含 “warn”级别的所有 lint。“warnings”组会忽略属性的顺序，并对实体执行所有警告lint 检查。

```rust,compile_fail
# unsafe fn an_unsafe_fn() {}
// 这里的两个属性的前后顺序无所谓
#[deny(warnings)]
// unsafe_code lint 默认是 "allow" 的。
#[warn(unsafe_code)]
fn example_err() {
    // 这里会编译报错，因为 `unsafe_code`警告的 lint 被提升为 "deny"级别了。
    unsafe { an_unsafe_fn() } // ERROR: usage of `unsafe` block
}
```

### Tool lint attributes
### 工具类lint属性


可以为 `allow`、`warn`、`deny` 或 `forbid` 这些调整代码检查级别的 lint属性添加/输入基于特定工具的 lint。注意该工具在当前作用域内可用才会起效。

工具类lint 只有在相关工具处于活动状态时才会做相应的代码模式检查。如果一个 lint级别属性，如 `allow`，引用了一个不存在的工具类lint，编译器将不会去警告不存在该 lint类，只有使用了该工具（，才会报告该 lint类不存在）。

在其他方面，这些工具类lint 就跟像常规的 lint 一样：

```rust
// 将 clippy 的整个 `pedantic` lint组设置为警告级别
#![warn(clippy::pedantic)]
// 使来自 clippy的 `filter_map` lint 的警告静音
#![allow(clippy::filter_map)]

fn main() {
    // ...
}

// 使 clippy的 `cmp_nan` lint 静音，仅用于此函数
#[allow(clippy::cmp_nan)]
fn foo() {
    // ...
}
```

> 注意：`rustc` 当前识别的工具类lint 有 "[clippy]" 和 "[rustdoc]"。

## The `deprecated` attribute
## `deprecated`属性

*`deprecated`属性*将程序项标记为已弃用。`rustc` 将对被 `#[deprecated]` 限定的程序项的行为发出警告。`rustdoc` 将特别展示已弃用的程序项，包括展示 `since` 版本和 `note` 提示（如果可用）。

`deprecated`属性有几种形式：

- `deprecated` --- 发布一个通用的弃用消息。
- `deprecated = "message"` --- 在弃用消息中包含给定的字符串。
- [_MetaListNameValueStr_]元项属性句法形式里带有两个可选字段：
  - `since` --- 指定程序项被弃用时的版本号。`rustc` 目前不解释此字符串，但是像 [Clippy] 这样的外部工具可以检查此值的有效性。
  - `note` --- 指定一个应该包含在弃用消息中的字符串。这通常用于提供关于不推荐的解释和推荐首选替代。

`deprecated`属性可以应用于任何[程序项][item]，[trait项][trait item]，[枚举变体][enum variant]，[结构体字段][struct field]，[外部块项][external block item]，或[宏定义][macro definition]。它不能应用于 [trait实现项][trait implementation items]。当应用于包含其他程序项的程序项时，如[模块][module]或[实现][implementation]，所有子程序项都继承此 deprecation属性。

<!-- 注意: 它只被 trait实现(AnnotationKind::Prohibited)拒绝。在这些之外的所有其他位置，它会被静默忽略。应用于元组结构体的字段时，此属性直接也被忽略。-->

这有个例子：

```rust
#[deprecated(since = "5.2.0", note = "foo was rarely used. Users should instead use bar")]
pub fn foo() {}

pub fn bar() {}
```

[RFC][1270-deprecation.md] 内有动机说明和更多细节。

[1270-deprecation.md]: https://github.com/rust-lang/rfcs/blob/master/text/1270-deprecation.md

## The `must_use` attribute
## `must_use`属性

*`must_use`属性* 用于在值未被“使用”时发出诊断警告。它可以应用于用户定义的复合类型（[结构体(`struct`)][struct]、[枚举(`enum`)][enum] 和 [联合体(`union`)][union]）、[函数][functions]和 [trait][traits]。

`must_use`属性可以使用[_MetaNameValueStr_]元项属性句法添加一些附加消息，如 `#[must_use = "example message"]`。该字符串将出现在警告消息里。

当用户定义的复合类型上使用了此属性，如果有该类型的[表达式][expression]正好是[表达式语句][expression statement]的表达式，那就违反了 `unused_must_use` 这个 lint。

```rust
#[must_use]
struct MustUse {
    // 一些字段
}

# impl MustUse {
#   fn new() -> MustUse { MustUse {} }
# }
#
// 违反 `unused_must_use` lint。
MustUse::new();
```

当函数上使用了此属性，如果此函数被当作[表达式语句][expression statement]的[表达式][expression]的[调用表达式][call expression]，那就违反了 `unused_must_use` lint。

```rust
#[must_use]
fn five() -> i32 { 5i32 }

// 违反 unused_must_use lint。
five();
```

当在 [trait声明][trait declaration]中使用了此属性时，如果[表达式语句][expression statement]的[调用表达式][call expression]返回了此 trait 的 [trait实现(impl trait)][impl trait]或 [trait对象（dyn trait）][dyn trait] ，那这种情况也违反 `unused_must_use` lint。

```rust
#[must_use]
trait Critical {}
impl Critical for i32 {}

fn get_critical() -> impl Critical {
    4i32
}

// 违反 `unused_must_use` lint。
get_critical();
```

当 trait声明中的一函数上使用了此属性时，如果调用表达式是此 trait 的某个实现中的该函数时，该行为同样违反 `unused_must_use` lint。

```rust
trait Trait {
    #[must_use]
    fn use_me(&self) -> i32;
}

impl Trait for i32 {
    fn use_me(&self) -> i32 { 0i32 }
}

// 违反 `unused_must_use` lint。
5i32.use_me();
```

当在 trait实现里的函数上使用 `must_use`属性时，此属性将被忽略。

> 注意：包含了此（属性应用的程序项产生的值）的普通空操作(no-op)表达式不会违反该 lint。例如，将此类值包装在没有实现 [`Drop`] 的类型中，然后不使用该类型，并成为未使用的[块表达式][block expression]的最终表达式(final expression)。
>
> ```rust
> #[must_use]
> fn five() -> i32 { 5i32 }
>
> // 这些都不违反 unused_must_use lint。
> (five(),);
> Some(five());
> { five() };
> if true { five() } else { 0i32 };
> match true {
>     _ => five()
> };
> ```

> 注意：当一个必须使用的值被故意丢弃时，使用带有模式 `_` 的[let语句][let statement]是惯用的方法。
>
> ```rust
> #[must_use]
> fn five() -> i32 { 5i32 }
>
> // 不违反 unused_must_use lint。
> let _ = five();
> ```

## The `diagnostic` tool attribute namespace
## `diagnostic`属性的命名空间

属性`#[diagnostic]` 的命名空间是影响编译期错误消息的根属性。
Rust 不保证这些属性提供的提示信息会被使用。
此命名空间可接受未知属性，但这样可能会触发有属性未被使用这样的警告。
此外，对已知属性的无效输入通常只是一个警告（有关详细信息，请参阅属性定义）。
这意味着允许添加或丢弃属性，并在未来更改输入，以允许进行更改，而无需为了保持程序正常工作而不得不选择无意义的属性或选项。

### The `diagnostic::on_unimplemented` attribute
### `diagnostic::on_unimplemented`属性 

`#[diagnostic::on_unimplemented]`属性会给编译器的一个提示，该提示通常用于要求某trait 必须在场，但使用的具体类型上并未实现此trait 的场景中来生成错误消息。
该属性应用在[trait申明][trait declaration]上，尽管它放在其他位置上并不会报错。
该属性使用[_MetaListNameValueStr_]元项属性句法来指定其输入，但该属性的任何格式错误的输入都不会被视为错误，此目的是为了提供向前和向后的兼容性。
其下的选项具有以下给定的含义：

* `message` --- 顶层错误消息的文本模板。
* `label` --- 错误消息中错误代码内联显示的文本标签。
* `note` --- 提供额外的注释。

其中 `note`选项可以出现多次，这将导致编译器发出多条注释消息。
如果其他选项中的任何一个出现了多次，则相关选项的首次定义会被认定为实际使用的值。
任何其他情况都会产生 lint 警告。
对于任何其他不存在的选项，也都会生成 lint 警告。

所有三个选项都接受字符串作为模板格式参数，使用与 [`std::fmt`] 字符串相同的格式进行解释。
使用给定命名参数的格式参数将被替换为以下文本：

* `{Self}` --- 实现 trait 的类型的名称。
* `{` *GenericParameterName* `}` --- 给定泛型参数的泛型参数的类型名。

任何其他格式参数都将生成警告，但会原样包含在字符串中。

无效的格式字符串可能会产生警告，但通常是被允许的，但可能不会按预期显示。
格式说明符可能会产生警告，但可以会被忽略。

示例：

```rust,compile_fail,E0277
#[diagnostic::on_unimplemented(
    message = "My Message for `ImportantTrait<{A}>` implemented for `{Self}`",
    label = "My Label",
    note = "Note 1",
    note = "Note 2"
)]
trait ImportantTrait<A> {}

fn use_my_trait(_: impl ImportantTrait<i32>) {}

fn main() {
    use_my_trait(String::new());
}
```

编译器可能会生成如下所示的错误消息：

```text
error[E0277]: My Message for `ImportantTrait<i32>` implemented for `String`
  --> src/main.rs:14:18
   |
14 |     use_my_trait(String::new());
   |     ------------ ^^^^^^^^^^^^^ My Label
   |     |
   |     required by a bound introduced by this call
   |
   = help: the trait `ImportantTrait<i32>` is not implemented for `String`
   = note: Note 1
   = note: Note 2
```

[Clippy]: https://github.com/rust-lang/rust-clippy
[_MetaListNameValueStr_]: ../attributes.md#meta-item-attribute-syntax
[_MetaListPaths_]: ../attributes.md#meta-item-attribute-syntax
[_MetaNameValueStr_]: ../attributes.md#meta-item-attribute-syntax
[`Drop`]: ../special-types-and-traits.md#drop
[attributes]: ../attributes.md
[block expression]: ../expressions/block-expr.md
[call expression]: ../expressions/call-expr.md
[dyn trait]: ../types/trait-object.md
[enum variant]: ../items/enumerations.md
[enum]: ../items/enumerations.md
[expression statement]: ../statements.md#expression-statements
[expression]: ../expressions.md
[external block item]: ../items/external-blocks.md
[functions]: ../items/functions.md
[impl trait]: ../types/impl-trait.md
[implementation]: ../items/implementations.md
[item]: ../items.md
[let statement]: ../statements.md#let-statements
[macro definition]: ../macros-by-example.md
[module]: ../items/modules.md
[rustc book]: https://doc.rust-lang.org/rustc/lints/index.html
[rustc-lint-caps]: https://doc.rust-lang.org/rustc/lints/levels.html#capping-lints
[rustc-lint-cli]: https://doc.rust-lang.org/rustc/lints/levels.html#via-compiler-flag
[rustdoc]: https://doc.rust-lang.org/rustdoc/lints.html
[struct field]: ../items/structs.md
[struct]: ../items/structs.md
[trait declaration]: ../items/traits.md
[trait implementation items]: ../items/implementations.md#trait-implementations
[trait item]: ../items/traits.md
[traits]: ../items/traits.md
[union]: ../items/unions.md
