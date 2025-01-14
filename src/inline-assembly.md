# Inline assembly
# 内联汇编

>[behavior-considered-undefined.md](https://github.com/rust-lang/reference/blob/master/src/inline-assembly.md)\
>commit: f1a0881bc0007dde9468171dd48bbddbdaddb28c \
>本章译文最后维护日期：2024-10-13

r[asm]

r[asm.intro]
Rust 通过 [`asm!`] 和 [`global_asm!`] 这两个宏来提供了对内联汇编的支持。
它可用于在编译器生成的汇编程序输出中嵌入手写的汇编程序。

[`asm!`]: core::arch::asm
[`global_asm!`]: core::arch::global_asm

r[asm.stable-targets]
目前对内联汇编的支持在以下目标架构上是稳定的：
- x86 和 x86-64
- ARM
- AArch64
- RISC-V
- LoongArch

如果在不支持的目标架构上使用 `asm!`，编译器会报错。

## Example
## 示例

r[asm.example]

```rust
# #[cfg(target_arch = "x86_64")] {
use std::arch::asm;

// 使用以为和相加运算来实现 x 乘6的效果
let mut x: u64 = 4;
unsafe {
    asm!(
        "mov {tmp}, {x}",
        "shl {tmp}, 1",
        "shl {x}, 2",
        "add {x}, {tmp}",
        x = inout(reg) x,
        tmp = out(reg) _,
    );
}
assert_eq!(x, 4 * 6);
# }
```

## Syntax
## 句法

r[asm.syntax]

下面的 ABNF语法规范指定了通用的内联汇编语法：

```text
format_string := STRING_LITERAL / RAW_STRING_LITERAL
dir_spec := "in" / "out" / "lateout" / "inout" / "inlateout"
reg_spec := <register class> / "\"" <explicit register> "\""
operand_expr := expr / "_" / expr "=>" expr / expr "=>" "_"
reg_operand := [ident "="] dir_spec "(" reg_spec ")" operand_expr / sym <path> / const <expr>
clobber_abi := "clobber_abi(" <abi> *("," <abi>) [","] ")"
option := "pure" / "nomem" / "readonly" / "preserves_flags" / "noreturn" / "nostack" / "att_syntax" / "raw"
options := "options(" option *("," option) [","] ")"
operand := reg_operand / clobber_abi / options
asm := "asm!(" format_string *("," format_string) *("," operand) [","] ")"
global_asm := "global_asm!(" format_string *("," format_string) *("," operand) [","] ")"
```

## Scope
## 作用域

r[asm.scope]

r[asm.scope.intro]
内联汇编可以以下面两种方式来使用。

r[asm.scope.asm]
通过 `asm!`宏，汇编代码在函数作用域中被发射，且最终被集成到编译器生成的函数的汇编代码中。
此汇编代码必须遵守 [严格规则](#rules-for-inline-assembly)以避免未定义行为。
注意，在某些情况下，编译器可能选择将汇编代码作为一个独立的函数发射，并生成对它的调用汇编。

r[asm.scope.global_asm]
通过 `global_asm!`宏，汇编代码在全局作用域中被发射。
这可以用来使用汇编代码编写完整的函数，并且通常提供更多的自由来使用任意寄存器和汇编指令。

## Template string arguments
## 模板字符串参数

r[asm.ts-args]

r[asm.ts-args.syntax]
汇编器模板使用与[格式字符串][format-syntax]相同的语法(即占位符是由一对花括号来标定)。

r[asm.ts-args.order]
相应的参数通过顺序、索引或名称来获取。

r[asm.ts-args.no-implicit]
但是不支持隐式的命名参数（[RFC #2795][rfc-2795]引入）。

r[asm.ts-args.one-or-more]
`asm!`宏调用可以有一个或多个模板字符串参数；带有多个模板字符串参数的 `asm!` 的所有字符串参数都会被视为用 `\n` 连接在了一起的大的字符串。
预期的用法是每个模板字符串参数对应一行汇编代码。

r[asm.ts-args.before-other-args]
所有的模板字符串参数必须放在其他类型的参数之前。

r[asm.ts-args.positional-first]
和格式字符串一样，命名参数必须在位置参数之后出现。
与格式字符串一样，位置参数必须出现在命名参数和显式[寄存器操作](#register-operands)之前。

r[asm.ts-args.register-operands]
在模板字符串中，占位符不能使用（带有）显式寄存器的操作。

r[asm.ts-args.at-least-once]
所有其他命名操作和位置操作必须在模板字符串中至少出现一次，否则将生成编译器错误。

r[asm.ts-args.opaque]
确切的汇编代码语法是特定于目标架构的，对于编译器来说是不透明的，当然除了编译器可以将各种操作类型去替换模板字符串以形成能传递给汇编器的汇编代码。

r[asm.ts-args.llvm-syntax]
目前，所有支持的目标架构都遵循 LLVM 内部汇编器使用的汇编代码语法，这通常对应于 GNU汇编器(GAS)。
x86目标架构上，默认使用 GAS的 `.intel_syntax noprefix`模式。
ARM目标架构上，使用 `.syntax unified`模式。
这些目标架构对汇编代码做了一些额外的限制：任何汇编器状态（例如，可以使用 `.section` 更改的当前节）必须在 asm字符串末尾恢复为其原始值。
不符合 GAS语法的汇编代码将导致特定于汇编器的行为。
对内联汇编使用的指令的进一步的约束在本章后面的[指令支持](#directives-support)章节有详细说明。

[format-syntax]: std::fmt#syntax
[rfc-2795]: https://github.com/rust-lang/rfcs/pull/2795

## Operand type
## 操作类型

r[asm.operand-type]

r[asm.operand-type.supported-operands]
目前支持以下几种类型的操作：

r[asm.operand-type.supported-operands.in]
* `in(<reg>) <expr>`
  - `<reg>` 可以指向某类寄存器或一个显式的寄存器。
    此已分配的寄存器名会被替换到 asm模板字符串中。
  - 在这段 asm代码的开头，此已分配的寄存器将包含 `<expr>` 的值。
  - 在这段 asm代码的结尾，该寄存器内的值必须恢复如初（除了另有 `lateout`操作也被分配使用了此寄存器）。  

r[asm.operand-type.supported-operands.out]
* `out(<reg>) <expr>`
  - `<reg>` 可以指向某类寄存器或一个显式的寄存器。
    此已分配的寄存器名会被替换到 asm模板字符串中。
  - 在这段 asm代码的开头，此已分配的寄存器将包含一个未定义的值。
  - `<expr>` 必须是一个（可能未被初始化的）位置表达式，在这段 asm代码的结尾，此寄存器的内容会被写到此位置表达式里。
  - 可以使用下划线(`_`)替代这个位置表达式，这将导致在这段 asm代码的结尾，此寄存器的内容被丢弃(等效于一个 clobber寄存器（译者注：clobber寄存器会被该asm语句中的汇编代码隐性修改，也因此，编译器在为输入操作数和输出操作数挑选寄存器时，就不会使用这类寄存器，这样就避免了发生数据覆盖等逻辑错误)。

r[asm.operand-type.supported-operands.lateout]
* `lateout(<reg>) <expr>`
  - 除了寄存器分配器可以重用分配给 `in` 的寄存器外，其他的同 `out`。
  - 应该只在读取所有输入后才写入此寄存器，否则可能会毁坏真实的输入。

r[asm.operand-type.supported-operands.inout]
* `inout(<reg>) <expr>`
  - `<reg>` 可以指向某类寄存器或一个显式的寄存器。
    此已分配的寄存器名会被替换到 asm模板字符串中。
  - 在这段 asm代码的开头，此已分配的寄存器将包含 `<expr>` 的值。
   - `<expr>` 必须是一个可变的已初始化的位置表达式，在这段 asm代码的结尾，此已分配的寄存器内的内容将会被写入此表达式。

r[asm.operand-type.supported-operands.inout-arrow]
* `inout(<reg>) <in expr> => <out expr>`
  - 除了从 `<in expr>` 里取值来初始化此寄存器外，其他的同 `inout`。
  - `<out expr>` 必须是一个（可能未被初始化的）位置表达式，在这段 asm代码的结尾，此寄存器的内容会被写到此位置表达式里。
  - 可以使用下划线(`_`)替代这个 `<out expr>`表达式，这将导致在这段 asm代码的结尾，此寄存器的内容被丢弃(等效于一个 clobber寄存器)。
  - `<in expr>` 和 `<out expr>` 可以有不同的类型。

r[asm.operand-type.supported-operands.inlateout]
* `inlateout(<reg>) <expr>` / `inlateout(<reg>) <in expr> => <out expr>`
  - 除了寄存器分配器可以重用分配给 `in` 的寄存器外（如果编译器知道 `in` 与 `inlateout` 具有相同的初始值，则可能发生这种情况），其他的同 `inout`。
  - 应该只在读取所有输入后才写入此寄存器，否则可能会毁坏真实的输入。

r[asm.operand-type.supported-operands.sym]
* `sym <path>`
  - `<path>` 必须指向一个 `fn`程序项 或 `static`程序项。
  - 引用该程序项的混淆符号名（mangled symbol）称被替换为 asm模板字符串。
  - 替换的字符串不包含任何修饰符（例如GOT、PLT、重定位等）。
  - `<path>`允许指向`#[thread_local]`静态项，在这种情况下，asm代码可以将符号与重定位（例如`@plt`、`@TPOFF`）结合起来，来从线程内的本地变量中读取数据。

* `const <expr>`
  - `<expr>`必须是整型常量表达式。此表达式遵循与内联 `const`块相同的规则。
  - 表达式的类型可以是任何整数类型，但默认为 `i32`，就像整型字面量一样。
  - 表达式的值被格式化为字符串，并直接替换为 asm模板字符串。

r[asm.operand-type.left-to-right]
操作表达式从左到右求值，就像函数调用参数一样。
在 `asm!`执行后，会按从左到右的顺序写出输出。
这一点很重要，如果两个输出指向同一个位置：该位置（表达式）将包含最右侧输出的值。

r[asm.operand-type.global_asm-restriction]
因为 `global_asm!` 存在于函数外部，它只能使用 `sym`操作 和 `const`操作。

## Register operands
## 寄存器操作

r[asm.register-operands]

r[asm.register-operands.register-or-class]
输入和输出操作可以被用来指定一个显式寄存器或某一类寄存器（可以被寄存器分配器选择和分配的一类寄存器，每次可以从中选择和分配其中的一个）。
显式寄存器是被字符串文本（例如 `"eax"`）指定的，而寄存器类是通过标识符（例如 `reg`）来指定的。

r[asm.register-operands.equivalence-to-base-register]
请注意，显式寄存器将寄存器别名（例如ARM上的 `r14` 和 `lr`）和寄存器的较小视图（例如 `eax` 和 `rax`）视为与其基寄存器(base register)等效。

r[asm.register-operands.error-two-operands]
对两个输入操作或两个输出操作使用相同的显式寄存器会在编译期报错。

r[asm.register-operands.error-overlapping]
此外，在输入操作或输出操作中发生寄存器重叠（如ARM VFP）也会在编译期报错。

r[asm.register-operands.allowed-types]
仅允许以下类型的值作为的内联汇编操作：
- 整型 (有符号的和无符号的)
- 浮点型数值
- 指针 (仅廋指针)
- 函数指针
- SIMD向量 (使用 `#[repr(simd)]` 定义的并且实现了 `Copy` 的结构体)。
这包括在 `std::arch` 中定义的特定于体系架构的向量类型，如 `__m128` (x86) 或 `int8x16_t` (ARM)。

r[asm.register-operands.supported-register-classes]
以下是当前支持的寄存器类列表：

|   体系架构   | 寄存器类 | 寄存器 | LLVM约束代码 |
| ------------ | -------------- | --------- | -------------------- |
| x86 | `reg` | `ax`, `bx`, `cx`, `dx`, `si`, `di`, `bp`, `r[8-15]` (仅 x86-64) | `r` |
| x86 | `reg_abcd` | `ax`, `bx`, `cx`, `dx` | `Q` |
| x86-32 | `reg_byte` | `al`, `bl`, `cl`, `dl`, `ah`, `bh`, `ch`, `dh` | `q` |
| x86-64 | `reg_byte`\* | `al`, `bl`, `cl`, `dl`, `sil`, `dil`, `bpl`, `r[8-15]b` | `q` |
| x86 | `xmm_reg` | `xmm[0-7]` (x86) `xmm[0-15]` (x86-64) | `x` |
| x86 | `ymm_reg` | `ymm[0-7]` (x86) `ymm[0-15]` (x86-64) | `x` |
| x86 | `zmm_reg` | `zmm[0-7]` (x86) `zmm[0-31]` (x86-64) | `v` |
| x86 | `kreg` | `k[1-7]` | `Yk` |
| x86 | `kreg0` | `k0` | 仅 clobbers |
| x86 | `x87_reg` | `st([0-7])` | 仅 clobbers |
| x86 | `mmx_reg` | `mm[0-7]` | 仅 clobbers |
| x86-64 | `tmm_reg` | `tmm[0-7]` | 仅 clobbers |
| AArch64 | `reg` | `x[0-30]` | `r` |
| AArch64 | `vreg` | `v[0-31]` | `w` |
| AArch64 | `vreg_low16` | `v[0-15]` | `x` |
| AArch64 | `preg` | `p[0-15]`, `ffr` | 仅 clobbers |
| ARM (ARM/Thumb2) | `reg` | `r[0-12]`, `r14` | `r` |
| ARM (Thumb1) | `reg` | `r[0-7]` | `r` |
| ARM | `sreg` | `s[0-31]` | `t` |
| ARM | `sreg_low16` | `s[0-15]` | `x` |
| ARM | `dreg` | `d[0-31]` | `w` |
| ARM | `dreg_low16` | `d[0-15]` | `t` |
| ARM | `dreg_low8` | `d[0-8]` | `x` |
| ARM | `qreg` | `q[0-15]` | `w` |
| ARM | `qreg_low8` | `q[0-7]` | `t` |
| ARM | `qreg_low4` | `q[0-3]` | `x` |
| RISC-V | `reg` | `x1`, `x[5-7]`, `x[9-15]`, `x[16-31]` (non-RV32E) | `r` |
| RISC-V | `freg` | `f[0-31]` | `f` |
| RISC-V | `vreg` | `v[0-31]` | 仅 clobbers |
| LoongArch | `reg` | `$r1`, `$r[4-20]`, `$r[23,30]` | `r` |
| LoongArch | `freg` | `$f[0-31]` | `f` |

> **注意**:
> - 在x86上，我们将 `reg_byte` 与 reg` 区别对待，因为编译器可以为 `reg_byte` 分别分配低位的 'al' 和高位的 'ah'，而 `reg` 却只能持有整个寄存器。
> - 在x86-64上，`reg_byte`寄存器类中没有高位寄存器（例如没有 `ah`）。
> - 一些寄存器类被标记为 "仅 clobbers" 意味着此类中的寄存器不能被用于输入或输出，只能用于 `out(<explicit register>) _` 或 `lateout(<explicit register>) _` 这样的表达形式的。

r[asm.register-operands.value-type-constraints]
每个寄存器类都有可以与之一起使用的值类型的约束。
这是必要的，因为值被加载到寄存器的方式取决于它的类型。
例如，在大端机系统上，将 `i32x4` 和 `i8x16` 加载到 SIMD寄存器可能会导致不同的寄存器内容，即便这两个值的字节内存表示形式相同。
特定寄存器类支持的类型的可用性可能取决于当前启用的目标特性。

| 体系架构 | 寄存器类 | 目标特性 | 允许的类型 |
| ------------ | -------------- | -------------- | ------------- |
| x86-32 | `reg` | None | `i16`, `i32`, `f32` |
| x86-64 | `reg` | None | `i16`, `i32`, `f32`, `i64`, `f64` |
| x86 | `reg_byte` | None | `i8` |
| x86 | `xmm_reg` | `sse` | `i32`, `f32`, `i64`, `f64`, <br> `i8x16`, `i16x8`, `i32x4`, `i64x2`, `f32x4`, `f64x2` |
| x86 | `ymm_reg` | `avx` | `i32`, `f32`, `i64`, `f64`, <br> `i8x16`, `i16x8`, `i32x4`, `i64x2`, `f32x4`, `f64x2` <br> `i8x32`, `i16x16`, `i32x8`, `i64x4`, `f32x8`, `f64x4` |
| x86 | `zmm_reg` | `avx512f` | `i32`, `f32`, `i64`, `f64`, <br> `i8x16`, `i16x8`, `i32x4`, `i64x2`, `f32x4`, `f64x2` <br> `i8x32`, `i16x16`, `i32x8`, `i64x4`, `f32x8`, `f64x4` <br> `i8x64`, `i16x32`, `i32x16`, `i64x8`, `f32x16`, `f64x8` |
| x86 | `kreg` | `avx512f` | `i8`, `i16` |
| x86 | `kreg` | `avx512bw` | `i32`, `i64` |
| x86 | `mmx_reg` | N/A | 仅 clobbers |
| x86 | `x87_reg` | N/A | 仅 clobbers |
| x86 | `tmm_reg` | N/A | 仅 clobbers |
| AArch64 | `reg` | None | `i8`, `i16`, `i32`, `f32`, `i64`, `f64` |
| AArch64 | `vreg` | `neon` | `i8`, `i16`, `i32`, `f32`, `i64`, `f64`, <br> `i8x8`, `i16x4`, `i32x2`, `i64x1`, `f32x2`, `f64x1`, <br> `i8x16`, `i16x8`, `i32x4`, `i64x2`, `f32x4`, `f64x2` |
| AArch64 | `preg` | N/A | 仅 clobbers |
| ARM | `reg` | None | `i8`, `i16`, `i32`, `f32` |
| ARM | `sreg` | `vfp2` | `i32`, `f32` |
| ARM | `dreg` | `vfp2` | `i64`, `f64`, `i8x8`, `i16x4`, `i32x2`, `i64x1`, `f32x2` |
| ARM | `qreg` | `neon` | `i8x16`, `i16x8`, `i32x4`, `i64x2`, `f32x4` |
| RISC-V32 | `reg` | None | `i8`, `i16`, `i32`, `f32` |
| RISC-V64 | `reg` | None | `i8`, `i16`, `i32`, `f32`, `i64`, `f64` |
| RISC-V | `freg` | `f` | `f32` |
| RISC-V | `freg` | `d` | `f64` |
| RISC-V | `vreg` | N/A | 仅 clobbers |
| LoongArch64 | `reg` | None | `i8`, `i16`, `i32`, `i64`, `f32`, `f64` |
| LoongArch64 | `freg` | None | `f32`, `f64` |

> **注意**: 对于上述表里，指针、函数指针和 `isize`/`usize` 被视为等效的整数类型（`i16`/`i32`/`i64` 具体取决于目标架构）。

r[asm.register-operands.smaller-value]
如果某个值的内存宽度小于分配给它的寄存器，则在输入时该寄存器的高位将有一个未定义的输入值，输出时将忽略该寄存器的高位值。
唯一的例外是 RISC-V 上的 `freg`寄存器类，其中 `f32` 值按照 RISC-V 体系架构的要求被以 NaN-boxed的形式表达为 `f64`。

r[asm.register-operands.separate-input-output]
当为 `inout`操作分别指定了输入和输出表达式时，这两个表达式必须具有相同的数据类型。
唯一的例外是两个操作（所操作的表达式）都是指针或整数时，它们只需要具有相同的类型内存宽度。
存在此约束是因为 LLVM 和 GCC 中的寄存器分配器有时无法让既定的操作(operand)和不同的数据类型进行有效绑定。

## Register names
## 寄存器名称

r[asm.register-names]

r[asm.register-names.supported-register-aliases]
一些寄存器有多个名称。
编译器会将它们视为与其基寄存器名称相同。
以下是所有受支持的寄存器别名的列表：

| 体系架构 | 基寄存器 | 别名 |
| ------------ | ------------- | ------- |
| x86 | `ax` | `eax`, `rax` |
| x86 | `bx` | `ebx`, `rbx` |
| x86 | `cx` | `ecx`, `rcx` |
| x86 | `dx` | `edx`, `rdx` |
| x86 | `si` | `esi`, `rsi` |
| x86 | `di` | `edi`, `rdi` |
| x86 | `bp` | `bpl`, `ebp`, `rbp` |
| x86 | `sp` | `spl`, `esp`, `rsp` |
| x86 | `ip` | `eip`, `rip` |
| x86 | `st(0)` | `st` |
| x86 | `r[8-15]` | `r[8-15]b`, `r[8-15]w`, `r[8-15]d` |
| x86 | `xmm[0-31]` | `ymm[0-31]`, `zmm[0-31]` |
| AArch64 | `x[0-30]` | `w[0-30]` |
| AArch64 | `x29` | `fp` |
| AArch64 | `x30` | `lr` |
| AArch64 | `sp` | `wsp` |
| AArch64 | `xzr` | `wzr` |
| AArch64 | `v[0-31]` | `b[0-31]`, `h[0-31]`, `s[0-31]`, `d[0-31]`, `q[0-31]` |
| ARM | `r[0-3]` | `a[1-4]` |
| ARM | `r[4-9]` | `v[1-6]` |
| ARM | `r9` | `rfp` |
| ARM | `r10` | `sl` |
| ARM | `r11` | `fp` |
| ARM | `r12` | `ip` |
| ARM | `r13` | `sp` |
| ARM | `r14` | `lr` |
| ARM | `r15` | `pc` |
| RISC-V | `x0` | `zero` |
| RISC-V | `x1` | `ra` |
| RISC-V | `x2` | `sp` |
| RISC-V | `x3` | `gp` |
| RISC-V | `x4` | `tp` |
| RISC-V | `x[5-7]` | `t[0-2]` |
| RISC-V | `x8` | `fp`, `s0` |
| RISC-V | `x9` | `s1` |
| RISC-V | `x[10-17]` | `a[0-7]` |
| RISC-V | `x[18-27]` | `s[2-11]` |
| RISC-V | `x[28-31]` | `t[3-6]` |
| RISC-V | `f[0-7]` | `ft[0-7]` |
| RISC-V | `f[8-9]` | `fs[0-1]` |
| RISC-V | `f[10-17]` | `fa[0-7]` |
| RISC-V | `f[18-27]` | `fs[2-11]` |
| RISC-V | `f[28-31]` | `ft[8-11]` |
| LoongArch | `$r0` | `$zero` |
| LoongArch | `$r1` | `$ra` |
| LoongArch | `$r2` | `$tp` |
| LoongArch | `$r3` | `$sp` |
| LoongArch | `$r[4-11]` | `$a[0-7]` |
| LoongArch | `$r[12-20]` | `$t[0-8]` |
| LoongArch | `$r21` | |
| LoongArch | `$r22` | `$fp`, `$s9` |
| LoongArch | `$r[23-31]` | `$s[0-8]` |
| LoongArch | `$f[0-7]` | `$fa[0-7]` |
| LoongArch | `$f[8-23]` | `$ft[0-15]` |
| LoongArch | `$f[24-31]` | `$fs[0-7]` |

r[asm.register-names.not-for-io]
某些寄存器不能用于输入或输出操作：

| 体系架构 | 不支持的寄存器 | 原因 |
| ------------ | -------------------- | ------ |
| All | `sp` | 栈指针必须在 asm代码块末尾恢复为其原始值。|
| All | `bp` (x86), `x29` (AArch64), `x8` (RISC-V), `$fp` (LoongArch) | 帧指针不能用作输入或输出。|
| ARM | `r7` or `r11` | 在 ARM 上，帧指针可以是 `r7` 或 `r11`，具体取决于目标架构。帧指针不能用作输入或输出。|
| All | `si` (x86-32), `bx` (x86-64), `r6` (ARM), `x19` (AArch64), `x9` (RISC-V), `$s8` (LoongArch) | LLVM 在内部将其用作具有复杂栈帧的函数的“基指针”。|
| x86 | `ip` | 这是程序计数器，不是真正的寄存器。|
| AArch64 | `xzr` | 这是一个不能修改的常量零寄存器。 |
| AArch64 | `x18` | 这是一些 AArch64目标架构上的操作系统预留寄存器。 |
| ARM | `pc` | 这是程序计数器，不是真正的寄存器。|
| ARM | `r9` | 这是一些 ARM目标架构上的操作系统预留寄存器。|
| RISC-V | `x0` | 这是一个不能修改的常量零寄存器。 |
| RISC-V | `gp`, `tp` | 这些寄存器是预留的，不能用作输入或输出。|
| LoongArch | `$r0` or `$zero` | 这是一个不能修改的常量零寄存器。 |
| LoongArch | `$r2` or `$tp` | 这是为TLS保留的。 |
| LoongArch | `$r21` | 这是 ABI 所保留的。 |

r[asm.register-names.fp-bp-reserved]
帧指针和基指针寄存器(base pointer register)预留供 LLVM 内部使用。而 `asm!`语句不能显式去指定使用预留寄存器，但在某些情况下，LLVM 将为 `reg`操作分配其中一个预留寄存器。使用预留寄存器的汇编代码应该小心，因为 `reg`操作可能在使用相同的寄存器。

## Template modifiers
## 模板修饰符

r[asm.template-modifiers]

r[asm.template-modifiers.intro]
占位符可以通过在大括号中的 `:` 之后指定的修饰符进行扩充。
这些修饰符不影响寄存器分配，但会更改插入模板字符串时操作(operand)的格式化方式。

r[asm.template-modifiers.only-one]
每个模板占位符只允许一个修饰符。

r[asm.template-modifiers.supported-modifiers]
支持的修饰符是 LLVM（和 GCC）[asm模板参数修饰符][llvm-argmod]的子集，但不使用相同的字母代码。

| 体系架构 | 寄存器类 | 修饰符 | 输出示例 | LLVM修饰符 |
| ------------ | -------------- | -------- | -------------- | ------------- |
| x86-32 | `reg` | None | `eax` | `k` |
| x86-64 | `reg` | None | `rax` | `q` |
| x86-32 | `reg_abcd` | `l` | `al` | `b` |
| x86-64 | `reg` | `l` | `al` | `b` |
| x86 | `reg_abcd` | `h` | `ah` | `h` |
| x86 | `reg` | `x` | `ax` | `w` |
| x86 | `reg` | `e` | `eax` | `k` |
| x86-64 | `reg` | `r` | `rax` | `q` |
| x86 | `reg_byte` | None | `al` / `ah` | None |
| x86 | `xmm_reg` | None | `xmm0` | `x` |
| x86 | `ymm_reg` | None | `ymm0` | `t` |
| x86 | `zmm_reg` | None | `zmm0` | `g` |
| x86 | `*mm_reg` | `x` | `xmm0` | `x` |
| x86 | `*mm_reg` | `y` | `ymm0` | `t` |
| x86 | `*mm_reg` | `z` | `zmm0` | `g` |
| x86 | `kreg` | None | `k1` | None |
| AArch64 | `reg` | None | `x0` | `x` |
| AArch64 | `reg` | `w` | `w0` | `w` |
| AArch64 | `reg` | `x` | `x0` | `x` |
| AArch64 | `vreg` | None | `v0` | None |
| AArch64 | `vreg` | `v` | `v0` | None |
| AArch64 | `vreg` | `b` | `b0` | `b` |
| AArch64 | `vreg` | `h` | `h0` | `h` |
| AArch64 | `vreg` | `s` | `s0` | `s` |
| AArch64 | `vreg` | `d` | `d0` | `d` |
| AArch64 | `vreg` | `q` | `q0` | `q` |
| ARM | `reg` | None | `r0` | None |
| ARM | `sreg` | None | `s0` | None |
| ARM | `dreg` | None | `d0` | `P` |
| ARM | `qreg` | None | `q0` | `q` |
| ARM | `qreg` | `e` / `f` | `d0` / `d1` | `e` / `f` |
| RISC-V | `reg` | None | `x1` | None |
| RISC-V | `freg` | None | `f0` | None |
| LoongArch | `reg` | None | `$r1` | None |
| LoongArch | `freg` | None | `$f0` | None |

> **注意**:
> - 对于 ARM架构 `e` / `f`: 这将打印出 NEON quad（128位）寄存器的低双字或高双字寄存器名称。
> - 对于 x86架构: 对于没有修饰符的 `reg`，Rust 编译器与 GCC 的编译行为不同。
>   GCC 将根据操作值的类型推断修饰符，而 Rust 默认为完整寄存器宽度。
> - 对于 x86架构的 `xmm_reg`: LLVM修饰符 `x`、`t` 和 `g` 目前还未在 LLVM 中真正实现(目前它们仅被 GCC 支持)，但这应该将只是一个小变动。

r[asm.template-modifiers.smaller-value]
如前一节开头所述，传递小于寄存器宽度的输入值将导致寄存器的高位包含未定义的值。
如果内联asm 仅访问寄存器的低位，则这不是问题，这可以通过使用模板修饰符在 asm代码中使用子寄存器名称（例如，使用 `ax` 替代 `rax`）来实现。
由于这是一个容易犯的错误，编译器将建议在适当的情况下使用模板修饰符来给出输入值的类型。
但如果某个操作的所有引用都已包含了修饰符，则对该操作的警告将会被抑制。

[llvm-argmod]: http://llvm.org/docs/LangRef.html#asm-template-argument-modifiers

## ABI clobbers

r[asm.abi-clobbers]

r[asm.abi-clobbers.intro]
关键字`clobber_abi` 可用于将与之对应的一组默认的 clobber寄存器应用于 `asm!`块内。
这在调用具有特定调用约定的函数时，会根据需要自动插入必要的 clobber约束：如果调用约定没有在调用过程中完整保持寄存器的值，则会将 `lateout("...") _` 隐式添加到操作列表中 (其中 `...` 要用具体的寄存器名字替换掉)。

r[asm.abi-clobbers.many]
`clobber_abi` 可以指定任意多次。它将为所有指定了调用约定的寄存器集合中的所有单一寄存器都分配一个 clobber寄存器。

r[asm.abi-clobbers.must-specify]
当使用 `clobber_abi` 时，不允许把通用寄存器类作为输出寄存器类来指定：此时所有输出寄存器必须显式指定。

r[asm.abi-clobbers.explicit-have-precedence]
有 `clobber_abi` 时，把显式输出寄存器标记为输出寄存器的工作优先于给寄存器隐式插入 clobber标志：此时只有当该寄存器明确未用作输出时，才会为寄存器插入 clobber标记。

r[asm.abi-clobbers.supported-abis]
以下的 ABI 可与 `clobber_abi` 一起使用：

| 体系架构 | ABI名称 | Clobbered寄存器 |
| ------------ | -------- | ------------------- |
| x86-32 | `"C"`, `"system"`, `"efiapi"`, `"cdecl"`, `"stdcall"`, `"fastcall"` | `ax`, `cx`, `dx`, `xmm[0-7]`, `mm[0-7]`, `k[1-7]`, `st([0-7])` |
| x86-64 | `"C"`, `"system"` (Windows平台), `"efiapi"`, `"win64"` | `ax`, `cx`, `dx`, `r[8-11]`, `xmm[0-31]`, `mm[0-7]`, `k[1-7]`, `st([0-7])`, `tmm[0-7]`  |
| x86-64 | `"C"`, `"system"` (在非Windows平台上), `"sysv64"` | `ax`, `cx`, `dx`, `si`, `di`, `r[8-11]`, `xmm[0-31]`, `mm[0-7]`, `k[1-7]`, `st([0-7])`, `tmm[0-7]`  |
| AArch64 | `"C"`, `"system"`, `"efiapi"` | `x[0-17]`, `x18`\*, `x30`, `v[0-31]`, `p[0-15]`, `ffr` |
| ARM | `"C"`, `"system"`, `"efiapi"`, `"aapcs"` | `r[0-3]`, `r12`, `r14`, `s[0-15]`, `d[0-7]`, `d[16-31]` |
| RISC-V | `"C"`, `"system"`, `"efiapi"` | `x1`, `x[5-7]`, `x[10-17]`, `x[28-31]`, `f[0-7]`, `f[10-17]`, `f[28-31]`, `v[0-31]` |
| LoongArch | `"C"`, `"system"` | `$r1`, `$r[4-20]`, `$f[0-23]` |

> 注意：
> - 在 AArch64架构上，如果 `x18` 不是目标架构上的预留寄存器，则它仅被包括在 clobber寄存器列表中。

随着体系架构中新的寄存器的加入，在rustc中，每个 ABI 对应的 clobbered寄存器列表也会一并更新：这确保了当 LLVM 开始在其生成的代码中使用这些新寄存器时，`asm!` 里的 clobber寄存器的正确性也能得到保持。

## Options
## 可选项

r[asm.options]

r[asm.options.supported-options]
我们使用标志（Flag）技术来进一步影响内联汇编块的行为。
目前定义了以下可选项（标志）：

r[asm.options.supported-options.pure]
- `pure`：此 `asm!`块没有副作用，且最终必须返回，其输出仅取决于其直接输入（比如，输入的值本身，而不是它们指向的对象）或从内存读取的值（除非还设置了 `nomem`可选项）。
  这允许编译器执行此 `asm!`块的次数少于程序中指定的次数（例如，通过将其从循环中提出来），或者如果 `asm!`块没有输出的情况下，编译器甚至可能完全消除此段 asm代码。
  `pure`可选项必须与 `nomem` 或 `readonly` 可选项组合使用，否则会导致编译期错误。

r[asm.options.supported-options.nomem]
- `nomem`：此 `asm!`块不访问 `asm!`块外的任何内存。
  这允许编译器将修改过的全局变量的值缓存在跨当前 `asm!`块的寄存器中，因为它知道当前 `asm!`不会读取或写入它们。
  编译器还假定此 `asm!`块不执行与其他线程的任何类型的同步，例如通过 fence。

r[asm.options.supported-options.readonly]
- `readonly`：此 `asm!`块不在 `asm!`块外写入任何内存。
  这允许编译器将未修改的全局变量的值缓存在跨当前 `asm!`块的寄存器中，因为它知道它们不是由当前 `asm!`写入的。
  编译器还假定此 `asm!`块不执行与其他线程的任何类型的同步，例如通过 fence。

r[asm.options.supported-options.preserves_flags]
- `preserves_flags`：此 `asm！`块不修改标志寄存器（在后面章节的规则表中有定义）。
  这可以避免编译器在此 `asm!`块之后重新计算条件标志。

r[asm.options.supported-options.noreturn]
- `noreturn`：此 `asm!`块没有具体返回值，其返回类型被定义为 `!`(never)。
  如果执行超过 asm代码的末尾，则为未定义行为。(译者注：asm代码末尾应该有跳转，否则执行程序会顺序执行二进制指令，从而执行超出 asm代码块)
  有 `noreturn`可选项的 asm块的行为就像一个不返回的函数；需要注意的是，其作用域中的局部变量在被调用之前不会被销毁。

r[asm.options.supported-options.nostack]
- `nostack`：此 `asm！`块不会将数据推送到栈上，也不会写入栈的红色区域（red-zone）（有些目标架构会支持red-zone）。
  如果此可选项*未*使用，则保证栈指针为函数调用会（根据目标ABI）适当对齐。

r[asm.options.supported-options.att_syntax]
- `att_syntax`：此可选项仅在 x86架构上有效，并导致汇编器使用 GNU汇编器的 `.att_syntax prefix`模式。
  寄存器操作在被替换时会自动加上前导`%`。

r[asm.options.supported-options.raw]
- `raw`：这将导致模板字符串被解析为裸汇编字符串，对 `{` 和 `}` 也不做特殊处理。
  这在使用 `include_str!` 从外部文件中包含裸汇编代码时非常有用。

r[asm.options.checks]
编译器对可选项执行一些附加检查：

r[asm.options.checks.mutually-exclusive]
  - `nomem` 和 `readonly` 是互斥的：同时指定这两个选项会导致编译期错误。

r[asm.options.checks.pure]
  - 在没有输出或只有丢弃的输出(`_`)的 asm块上指定 `pure` 会导致编译期错误。

r[asm.options.checks.noreturn]
  - 在带有输出的 asm块上指定 `noreturn` 会导致编译期错误。

r[asm.options.global_asm-restriction]
`global_asm!` 仅支持 `att_syntax` 和 `raw` 可选项。
其余可选项对于全局作用域类型的内联汇编没有意义。

## Rules for inline assembly
## 内联汇编规则

r[asm.rules]

r[asm.rules.intro]
为了避免未定义行为，在使用函数作用域类型的内联汇编（`asm!`）时必须遵循以下规则：

r[asm.rules.reg-not-input]
- 任何未指定为输入的寄存器在进入 asm块时都会包含未定义的值。
  - 内联汇编上下文中的“未定义值”意味着寄存器可以（非确定性地）具有体系架构允许的任何一个可能值。
    值得注意的是，它与 LLVM 的 `undef` 不同，LLVM 的 `undef` 每次读取时都会有不同的值（因为在汇编代码中不存在这样的概念）。

r[asm.rules.reg-not-output]
- 任何未指定为输出的寄存器在退出 asm块时必须具有与进入时相同的值，否则为未定义行为。
  - 这仅适用于可指定为输入或输出的寄存器。
    其他寄存器遵循特定于目标架构的规则。
  - 请注意，`lateout` 可以分配给与 `in` 相同的寄存器，在这种情况下，此规则不适用。
    然而，代码不应该依赖于此，因为它依赖于寄存器分配的结果。

r[asm.rules.unwind]
- 如果在 asm块中执行展开（unwind），则为未定义行为。
  - 如果汇编代码调用一个函数，然后该函数展开，则这也适用此规则。

r[asm.rules.mem-same-as-ffi]
- 汇编代码允许读取和写入的内存位置集与 FFI函数允许读取和写入的内存位置集相同。
  - 有关确切规则，请参阅不安全代码指南。
  - 如果设置了 `readonly`可选项，则只允许内存读取。
  - 如果设置了 `nomem`可选项，则不允许对内存进行读取或写入。
  - 这些规则不适用于 asm代码私有的内存空间，例如 asm块内分配的堆栈空间。

r[asm.rules.black-box]
- 编译器不能假定 asm块中的指令是最终实际执行的指令。
  - 这实际上意味着编译器必须将 `asm!` 作为一个黑盒对待，只考虑接口规范，而不考虑 asm块内的指令本身。
  - 允许通过特定于目标架构的机制来执行运行时代码补丁。
  - 然而，不能保证每个 `asm！`直接对应于目标文件中指令的单个实例：编译器可以自由复制或消除重复的 `asm！`块。

r[asm.rules.stack-below-sp]
- 除非设置了 `nostack`可选项，否则可以允许 asm代码使用栈指针下方的栈空间。
  - 当进入 asm块后，为保证栈指针为函数调用，此指针会（根据目标ABI）适当对齐。
  - 你有责任确保不会溢出堆栈（例如，使用堆栈探测以确保命中保护页）。
  - 根据目标ABI的要求分配堆栈内存时，应调整堆栈指针。
  - 在离开 asm块之前，必须将堆栈指针还原为其原始值。

r[asm.rules.noreturn]
- 如果设置了 `noreturn`可选项，则如果执行超过 asm块的末尾，则此行为未定义。

r[asm.rules.pure]
- 如果设置了 `pure`可选项，那么如果 `asm!`代码除了有直接输出外，还有其他副作用，则此行为未定义。
  如果两次执行 `asm!`代码，它们具有相同输入，但输出不同，此行为也未定义。 
  - 当与 `nomem`可选项一起使用时，所谓的“输入”只能是 `asm!`的直接输入。
  - 当与 `readonly`可选项一起使用时，所谓的“输入”包含 `asm!`的直接输入，还包含允许 `asm!`块来读取的全部内存。

r[asm.rules.preserved-registers]
- 如果设置了 `preserves_flags`可选项，则在退出 asm块时必须恢复这些标志寄存器：
  - x86
    - `EFLAGS` (CF, PF, AF, ZF, SF, OF)中的状态标志寄存器。
    - 浮点状态字 (全部)。
    - `MXCSR` (PE, UE, OE, ZE, DE, IE)中的浮点异常标志寄存器。
  - ARM
    - `CPSR` (N, Z, C, V)中的条件标志寄存器。
    - `CPSR` (Q)中的饱和标志寄存器。
    - `CPSR` (GE)中的大于或等于标志寄存器。
    - `FPSCR` (N, Z, C, V)中的条件标志寄存器。
    - `FPSCR` (QC)中的饱和标志寄存器。
    - `FPSCR` (IDC, IXC, UFC, OFC, DZC, IOC)中的浮点异常标志寄存器。
  - AArch64
    - 条件标志寄存器(`NZCV`)。
    - 浮点状态寄存器(`FPSR`).
  - RISC-V
    - `fcsr` (`fflags`)中的浮点异常标志寄存器。
    - 向量扩展状态寄存器(`vtype`, `vl`, `vcsr`)。
  - LoongArch
    - `$fcc[0-7]` 中的浮点条件标志寄存器。

r[asm.rules.x86-df]
- 在 x86 上，方向标志寄存器（`EFLAGS` 中的 DF）在进入 asm块时清除，在退出时也必须清除。
  - 如果在退出 asm块时设置了方向标志，则此行为未定义。

r[asm.rules.x86-x87]
- 在 x86 上，x87浮点寄存器栈必须保持不变，除非所有的 `st([0-7])`寄存器都被用 `out("st(0)") _, out("st(1)") _, ...`标记标记为 clobber寄存器。
  - 如果所有 x87寄存器都被标记为 clobber寄存器，则在进入 `asm`块时，x87寄存器栈被保证为空。并且汇编代码必须确保退出 asm块时 x87寄存器栈也为空。

r[asm.rules.only-on-exit]
- 将堆栈指针和非输出寄存器恢复为其原始值的要求仅在退出 `asm!`块时适用。
  - 这意味着永不返回的 `asm!`块（即使未标记为 `noreturn`）不需要恢复这些寄存器。
  - 当返回到其他的 `asm!`块（例如上下文切换）时，这些寄存器必须包含它们在进入当前（你正在*退出*）的 `asm!`块的值。
    - 你不能退出尚未进入的 `asm!`块。
      你也不能退出已退出的 `asm!`块（也可以理解为不能未进入而退出）。
    - 你负责切换任何特定于目标架构的状态（例如线程级的本地存储和堆栈边界）。
    - 你不能从一个 `asm!`块中的地址跳转块到另一个 `asm!`块中的地址，即使在同一个函数或块中也不行。应将两个 `asm!`块的上下文视为潜在不同，跳转时会进行上下文切换。  你不能假设这些上下文中的任何特定值（例如当前堆栈指针或堆栈指针下的临时值）在两个 `asm!`块之间保持不变。
    - 你可以访问的内存位置集是你进入的 `asm!`块所允许的内存位置和你刚退出的 `asm!`块所允许的内存位置的交集。

r[asm.rules.not-successive]
- 即使两个 `asm!`块在源代码中相邻，并且它们之间没有任何其他代码，你仍不能假设它们也将在二进制中的地址连续，不能保证它们之间没有任何其他指令。

r[asm.rules.not-exactly-once]
- 你不能假设 `asm!`块在输出的二进制文件中只出现一次。
  允许编译器实例化 `asm!`块的多个副本块，例如当包含它的函数在多个位置内联时。

r[asm.rules.x86-prefix-restriction]
- 在 x86 上，内联汇编不能以指令前缀（如 `LOCK`）结尾（这些指令前缀被用于编译器生成的指令）。
  - 由于内联汇编的编译方式，编译器当前还无法检测到此问题，但将来可能会捕获并拒绝此问题。

r[asm.rules.preserves_flags]
> **注意**：作为一个一般性原则，`preserves_flags` 包含的标志是在执行函数调用时*未*保留的标志。

### Correctness and Validity
### 正确性和有效性

r[asm.validity]

r[asm.validity.necessary-but-not-sufficient]
除了前面的所有规则外，`asm!` 的字符串参数（在计算完所有其他参数后，执行格式化并转换其操作数）必须最终转换成为（对于目标体系结构而言）语法正确且语义有效的汇编代码。
其中，格式化规则允许编译器生成具有正确语法的汇编代码。
有关操作数的规则允许将 Rust操作数有效转换为`asm!`，以及从 `asm!` 转换出。
为使最终汇编代码既正确又有效，遵守这些规则是必要的，但还不够。例如：

- 格式化后，参数可能被放置在语法不正确的位置
- 一条指令可能被正确写入，但该操作数在给定的体系结构上却是无效的
- 对应体系结构未支持的指令可以汇编成不确定的代码
- 一组指令，每一条都正确有效，如果连续放置，仍有可能会导致未定义的行为

r[asm.validity.non-exhaustive]
因此，这些规则是 _非穷尽_ 的。编译器不需要检查初始字符串的正确性和有效性，也不需要检查生成的最终汇编代码。
汇编器可以检查这些代码的正确性和有效性，但不需要这样做。
使用 `asm!` 时，键入错误就足以使程序变得不健壮。完全排除这类错误太难了，汇编规则那是包含在数千页的体系结构参考手册中的呀。
程序员应该谨慎行事，调用这种 `unsafe` 的功能需要承担不违反编译器或体系结构规则的责任。

### Directives Support
### 伪指令支持

r[asm.directives]

r[asm.directives.subset-supported]
内联汇编支持 GNU AS 和 LLVM 的内部汇编器支持的伪指令集的一个子集，具体如下所示。
也有部分伪指令的效果是特定于汇编器的（可能会导致错误，或者可能会被接受）。

r[asm.directives.stateful]
如果内联汇编包含任何修改后续汇编程序的“有状态(stateful)”伪指令，则块必须在内联汇编结束之前撤消任何此类伪指令的执行效果。

r[asm.directives.supported-directives]
下面这些伪指令被汇编器确保支持：

- `.2byte`
- `.4byte`
- `.8byte`
- `.align`
- `.alt_entry`
- `.ascii`
- `.asciz`
- `.balign`
- `.balignl`
- `.balignw`
- `.bss`
- `.byte`
- `.comm`
- `.data`
- `.def`
- `.double`
- `.endef`
- `.equ`
- `.equiv`
- `.eqv`
- `.fill`
- `.float`
- `.global`
- `.globl`
- `.inst`
- `.insn`
- `.lcomm`
- `.long`
- `.octa`
- `.option`
- `.p2align`
- `.popsection`
- `.private_extern`
- `.pushsection`
- `.quad`
- `.scl`
- `.section`
- `.set`
- `.short`
- `.size`
- `.skip`
- `.sleb128`
- `.space`
- `.string`
- `.text`
- `.type`
- `.uleb128`
- `.word`


#### Target Specific Directive Support
#### 基于特定目标规范的指令支持

r[asm.target-specific-directives]

##### Dwarf Unwinding
##### Dwarf展开

r[asm.target-specific-directives.dwarf-unwinding]
支持 DWARF展开信息的 ELF目标平台支持以下指令：

- `.cfi_adjust_cfa_offset`
- `.cfi_def_cfa`
- `.cfi_def_cfa_offset`
- `.cfi_def_cfa_register`
- `.cfi_endproc`
- `.cfi_escape`
- `.cfi_lsda`
- `.cfi_offset`
- `.cfi_personality`
- `.cfi_register`
- `.cfi_rel_offset`
- `.cfi_remember_state`
- `.cfi_restore`
- `.cfi_restore_state`
- `.cfi_return_column`
- `.cfi_same_value`
- `.cfi_sections`
- `.cfi_signal_frame`
- `.cfi_startproc`
- `.cfi_undefined`
- `.cfi_window_save`

##### Structured Exception Handling
##### 结构化异常处理

r[asm.target-specific-directives.structured-exception-handling]
在带有结构化异常处理机制的目标平台上，可以确保下面的这些附加指令得到支持：

- `.seh_endproc`
- `.seh_endprologue`
- `.seh_proc`
- `.seh_pushreg`
- `.seh_savereg`
- `.seh_setframe`
- `.seh_stackalloc`


##### x86 (32-bit and 64-bit)
##### x86 (32位 and 64位)

r[asm.target-specific-directives.x86]
无论是 32位还是 64位 x86目标平台，可以确保下面的这些附加指令得到支持：
- `.nops`
- `.code16`
- `.code32`
- `.code64`

只有在退出汇编块之前将状态重置为默认状态时，才支持使用 `.code16`、`.code32` 和 `.code64` 这些指令。
默认情况下，32位 x86平台使用 `.code32`，64位 x86平台使用 `.code64`。

##### ARM (32-bit)
##### ARM (32位)

r[asm.target-specific-directives.arm-32-bit]
ARM目标平台下，可以确保下面的这些附加指令得到支持：

- `.even`
- `.fnstart`
- `.fnend`
- `.save`
- `.movsp`
- `.code`
- `.thumb`
- `.thumb_func`
