# Introduction
# 介绍

>[introduction.md](https://github.com/rust-lang/reference/blob/master/src/introduction.md)\
>commit: e7345c94d528c4c69e11b3966db43d9dcc964638 \
>本章译文最后维护日期：2024-10-13

本书是 Rust 编程语言的主要参考手册，本书提供了3类资料：
  - 一些章节非正式地介绍了该语言的各种语言结构及其用法。
  - 一些章节非正式地介绍了该语言的内存模型、并发模型、运行时服务、链接模型，以及调试工具。-
  - 附录章节提供了一些对 Rust 语言有影响的编程原理和参考资料。

> [!WARNING]

> 此书尚未完成，记录 Rust 的所有内容需要花些时间。
> 有关本书未记录的内容，请查阅 [GitHub issues]。

</div>

## Rust releases
## Rust 发行版

Rust 每六周发布一种新的版本。
该语言的第一个稳定版本是 Rust 1.0.0，然后是 Rust 1.1.0，以此类推。
相应的工具（`rustc`、’`cargo`，等）和文档([标准库][Standard library]、本书，等等)与语言版本一起发布。

本书的最新版本，与最新的 Rust 版本相匹配，它总是在 <https://doc.rust-lang.org/reference/> 等你。

通过在此链接的“reference”路径段之前添加相应的 Rust版本号，可以找到以往的版本。
比如，Rust 1.49.0版在 <https://doc.rust-lang.org/1.49.0/reference/> 下。

## What *The Reference* is Not
## *参考手册* 并非——

这本书不是对这门语言的入门介绍。本书假设您熟悉该语言。若您需要学习该语言的基础知识，请阅读 [Rust程序设计语言][book]。

这本书也不作为 Rust 语言发行版中包含的[标准库][standard library]的参考资料。Rust 的库文档是从其源代码文件中提取的文档属性。此外，有许多可能被可能认为是语言自带特性(features)的特性其实都是 Rust 的标准库的特性，所以您要寻找的特性可能在那里，而不是在这里。

类似地，本书通常不能作为记录 rustc 或者 Cargo 细节的工具书。rustc 有自己专门的书 [rustc book]，Cargo 也有一本书 [cargo book]，该书中包含了 Cargo 的[参考手册][cargo reference]。本书也涉及了少量和它们的相关知识，比如[链接][linkage]的章节，介绍了 rustc 是如何工作的。

本书仅作为稳定版 Rust 的参考资料存在，关于尚在开发中的非稳定特性，请参阅 [Unstable Book]。

Rust编译器（包括 `rustc`）将执行编译优化，但本参考手册不指导这些编译器的优化工作，所以只能把编译后的程序看作一个黑盒子。如果想侦测它的优化方式，只能通过运行它、给它提供输入并观察它的输出来推断。所有的编译器优化结果和程序的运行效果都必须和本参考手册所说的保持一致。

最后，本书并非 Rust 语言规范。它可能包含特定于 `rustc` 的细节，这不应该被当作 Rust 语言的规范。我们打算以后出版这样一本书，但在那之前，本手册是最接近的东西。

## How to Use This Book
## 如何使用此书

本书不会假定您是按顺序阅读本书。本书的每一章一般都可以独立阅读，但会交叉链接到其他章节，以了解它们所相关的内容，但不会进行讨论。

阅读本书有两种主要方式。

第一是寻找特定问题的答案。如果您知道回答问题的章节，您可以直接从目录跳入该章节进行阅读。否则，您可以按 `s` 键或单击顶部栏上的放大镜来搜索与问题相关的关键字（译者注：目前本翻译版还不支持这种搜索功能）。例如，假设您想知道在 `let`语句中创建的临时值何时被销毁；同时，假设您还不知道[表达式][expressions chapter]那一章中定义了[临时对象的生存期][lifetime of temporaries]，那么可以搜索 “temporary let” ，第一个搜索结果将带您去阅读该部分。

第二是提高您对此语言某一方面的认知。在这种情况下，只需浏览目录，直到看到您想了解的内容，然后点开阅读。阅读中如果某个链接看起来很有帮助，那就点击并阅读该部分内容。

也就是说，本书没有错误的阅读方式。您觉得怎样读对您最有帮助就怎样读。

### Conventions
### 约定

像所有技术书籍一样，在如何展示信息方面，本书有一些约定。这些约定记录如下。

* 定义术语的语句中，术语写为*斜体*。当术语在其定义章节之外使用时，通常会有一个链接来指向该术语定义的章节。

  *示例术语* 这是一个定义术语的示例。

* 编译 crate 所使用的版次之间的语言差异用一个块引用表示，以**粗体**的“版次差异：”开头。

  > **版次差异**：此句法(syntax)在 2015 版次有效，2018 版次起不允许使用。

* 一些有用信息，比如有关本书状态，或指出有用但大多超出本书范围的信息，大都位于以**粗体**的“注：”开头的块注释中。
  
  > **注**：这是一个注释示例。

* 有关对语言的不健全(sound)行为，或者针对易于混淆的语言特性的警告，记录在特殊的警告框里。

  > [!WARNING]

  > 这是一个示例警告。

* 文本中内联的代码片段在 `<code>` 标签里。

  较长的代码示例放在突出显示句法(syntax)的框中，该框的右上角有用于复制、执行和显示隐藏行的控件
  
  ```rust
  # // 这是隐藏行。
  fn main() {
      println!("这是一段示例代码。");
  }
  ```
  除非另有说明，否则所有示例均使用最新版本的语法和编译检查。

* 文法和词法结构放在块引用中，第一行为粗体上标的 <sup>**词法**</sup> 或 <sup>**句法**</sup>。

  > **<sup>句法</sup>**\
  > _ExampleGrammar_:\
  > &nbsp;&nbsp; &nbsp;&nbsp; `~` [_Expression_]\
  > &nbsp;&nbsp; | `box` [_Expression_]

  查阅[表义符(notation)][Notation]以获取更多细节。

* 规则标识符出现在用方括号括起的每个语言规则之前。这些标识符提供了一种在语言中引用特定规则的方法。规则标识符使用句点来分隔从最一般到最具体的部分（例如，[destructors.scope.nesting.function-body]）。

  可以单击规则名称以链接到该规则。
  
r[example.rule.label]

  > [!WARNING]
  > 规则的组织方式目前在不断变化。目前，这些标识符名称在各个版本之间并不稳定，如果更改了这些规则，则指向这些规则的链接可能会失败。一旦组织方式稳定下来，我们打算稳定这些，让规则名称的链接不会在版本之间失效。

## Contributing
## 贡献力量

我们欢迎各种形式的贡献。

您可以通过开启议题或向 [Rust 参考手册仓库][the Rust Reference repository]发送 PR 来为本书做出贡献。如果这本书没有回答您的问题，并且您认为它的答案应该在本书的范围内，请不要犹豫，[提交议题][file an issue]或在 [Zulip] 的 `t-lang/doc` 流频道上询问。知道人们最喜欢用这本书来做什么将有助于引导我们的注意力来让这些部分变得更好。我们也希望此手册尽可能地规范，所以如果你看到任何错误或非规范的地方，但没有明确指出，也请[提交议题][file an issue]。

[book]: https://doc.rust-lang.org/book/index.html
[github issues]: https://github.com/rust-lang/reference/issues
[standard library]: std
[the Rust Reference repository]: https://github.com/rust-lang/reference/
[Unstable Book]: https://doc.rust-lang.org/nightly/unstable-book/
[_Expression_]: expressions.md
[cargo book]: https://doc.rust-lang.org/cargo/index.html
[cargo reference]: https://doc.rust-lang.org/cargo/reference/index.html
[expressions chapter]: expressions.html
[file an issue]: https://github.com/rust-lang/reference/issues
[lifetime of temporaries]: expressions.html#temporaries
[linkage]: linkage.html
[rustc book]: https://doc.rust-lang.org/rustc/index.html
[Notation]: notation.md
[Zulip]: https://rust-lang.zulipchat.com/#narrow/stream/237824-t-lang.2Fdoc