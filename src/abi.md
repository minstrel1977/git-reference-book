# Application Binary Interface (ABI)
# 应用程序二进制接口(ABI)

>[abi.md.md](https://github.com/rust-lang/reference/blob/master/src/abi.md)\
>commit:  73c11acaf1d161fdf953eeba1be7cd2d0c0a79a8 \
>本章译文最后维护日期：2024-10-13

本节介绍影响 crate 编译输出的 ABI 的各种特性。

有关为导出函数(exporting functions)指定 ABI 的信息，请参阅[*外部函数*][extern functions]。参阅[*外部块*][external blocks]了解关于指定 ABI 来链接外部库的信息。

## The `used` attribute
## `used`属性

*`used`属性*只能用在[静态(`static`)项][`static` items]上。此[属性][attribute]强制编译器将该变量保留在输出对象文件中(.o、.rlib 等，不包括最终的二进制文件)，即使该变量没有被 crate 中的任何其他项使用或引用。注意，链接器(linker)仍有权移除此类变量。

下面的示例显示了编译器在什么条件下在输出对象文件中保留静态(`static`)项。

``` rust
// foo.rs

// 将保留，因为 `#[used]`:
#[used]
static FOO: u32 = 0;

// 可移除，因为没实际使用:
#[allow(dead_code)]
static BAR: u32 = 0;

// 将保留，因为这个是公有的:
pub static BAZ: u32 = 0;

// 将保留，因为这个被可达公有函数引用:
static QUUX: u32 = 0;

pub fn quux() -> &'static u32 {
    &QUUX
}

// 可移除，因为被私有且未被使用的函数引用:
static CORGE: u32 = 0;

#[allow(dead_code)]
fn corge() -> &'static u32 {
    &CORGE
}
```

``` console
$ rustc -O --emit=obj --crate-type=rlib foo.rs

$ nm -C foo.o
0000000000000000 R foo::BAZ
0000000000000000 r foo::FOO
0000000000000000 R foo::QUUX
0000000000000000 T foo::quux
```

## The `no_mangle` attribute
## `no_mangle`属性

可以在任何[程序项][item]上使用 *`no_mangle`属性*来禁用标准名称符号名混淆(standard symbol name mangling)。禁用此功能后，此程序项的导出符号(symbol)名将直接是此程序项的原来的名称标识符。

此外，就跟[`used`属性](#the-used-attribute)一样，此属性修饰的程序项也将从生成的库或对象文件中公开导出。

此属性是非安全属性，因为非混淆符号可能与另一个同名符号（或已知符号）冲突，从而导致未定义的行为。

```rust
#[unsafe(no_mangle)]
extern "C" fn foo() {}
```

## The `link_section` attribute
## `link_section`属性

`link_section`属性指定了输出对象文件中[函数][function]或[静态项][static]的内容将被放置到的节点位置。它使用 [_MetaNameValueStr_]元项属性句法指定节点名称。

此属性是非安全的，因为它允许用户将数据和代码放入不需要它们的内存代码段中，例如将可变数据放入只读代码段。

<!-- no_run: don't link. The format of the section name is platform-specific. -->
```rust,no_run
#[unsafe(no_mangle)]
#[unsafe(link_section = ".example_section")]
pub static VAR1: u32 = 1;
```

## The `export_name` attribute
## `export_name`属性

*`export_name`属性*指定将在[函数][function]或[静态项][static]上导出的符号的名称。它使用 [_MetaNameValueStr_]元项属性句法指定符号名。

此属性是非安全的，因为具有自定义名称的符号可能会与另一个具有相同名称的符号（或与已知符号）冲突，从而导致未定义行为。

```rust
#[unsafe(export_name = "exported_symbol_name")]
pub fn name_in_rust() { }
```

[_MetaNameValueStr_]: attributes.md#meta-item-attribute-syntax
[`static` items]: items/static-items.md
[attribute]: attributes.md
[extern functions]: items/functions.md#extern-function-qualifier
[external blocks]: items/external-blocks.md
[function]: items/functions.md
[item]: items.md
[static]: items/static-items.md