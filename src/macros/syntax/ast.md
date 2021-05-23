# AST 中的宏

如前所述，在 Rust 中，宏处理发生在 AST 生成之后。
因此，调用宏的语法 *必须* 是Rust语言语法中规整相符的一部分。

实际上，Rust 语法包含数种“语法扩展”的形式。具体来说有以下四种（顺便给出例子）：

* `# [ $arg ]` 形式的：`#[derive(Clone)]`, `#[no_mangle]`, …
* `# ! [ $arg ]` 形式的：`#![allow(dead_code)]`, `#![crate_name="blang"]`, …
* `$name ! $arg` 形式的：`println!("Hi!")`, `concat!("a", "b")`, …
* `$name ! $arg0 $arg1` 形式的：`macro_rules! dummy { () => {}; }`.

头两种形式被称作“属性” ([attributes])。属性用来给条目 (items) 、表达式、语句加上注解。

属性有三类：内置的属性 ([built-in attributes])、宏属性 ([macro attributes])、派生属性([derive attributes])。

内置的属性由编译器实现。宏属性和派生属性在 Rust 第二类宏系统——过程宏 ([procedural macros])——中实现。


我们感兴趣的是第三种：`$name ! $arg`——像函数那样使用的宏。
这种形式的宏 *可以* 用 `macro_rules!` 来声明。

注意第三种形式的宏是一种一般的语法拓展形式，并非仅用 `macro_rules!` 写出。
比如 [`format!`] 是一个宏，而用来实现 [`format!`] 的 [`format_args!`] 不是这里谈论的宏（后者由编译器实现）。

第四种形式实际上宏无法使用。事实上，这种形式的唯一用例只有 `macro_rules!`，我们将在稍后谈到它。

将注意力集中到第三种形式 `$name ! $arg` 上，我们的问题变成，对于每种可能的语法扩展，
Rust的语法解析器 (parser) 如何知道 `$arg` 究竟长什么样？

答案是它 *不需要* 知道。其实，提供给每次语法扩展调用的参数，是*一棵* 标记树 (token tree)。
具体来说，是一棵非叶节点的标记树： `(...)`、`[...]` 或 `{...}`。
知道这一点后，语法分析器如何理解如下调用形式，就变得显而易见了：

```rust,ignore
bitflags! {
    struct Color: u8 {
        const RED    = 0b0001,
        const GREEN  = 0b0010,
        const BLUE   = 0b0100,
        const BRIGHT = 0b1000,
    }
}

lazy_static! {
    static ref FIB_100: u32 = {
        fn fib(a: u32) -> u32 {
            match a {
                0 => 0,
                1 => 1,
                a => fib(a-1) + fib(a-2)
            }
        }

        fib(100)
    };
}

fn main() {
    use Color::*;
    let colors = vec![RED, GREEN, BLUE];
    println!("Hello, World!");
}
```

虽然上述调用 *看起来* 包含了各式各样的 Rust 代码，但对语法分析器来说，它们仅仅是堆毫无意义的标记树。
为了让事情变得更清晰，我们把所有这些句法“黑盒”用 ⬚ 代替，仅剩下：

```text
bitflags! ⬚

lazy_static! ⬚

fn main() {
    let colors = vec! ⬚;
    println! ⬚;
}
```
再次重申，语法分析器对 ⬚ 不作任何假设；它记录黑盒所包含的标记，但并不尝试理解它们。

以下几点很重要：

*   Rust包含多种语法扩展。我们将仅仅讨论定义在 `macro_rules!` 结构中的宏。
*   当遇见形如 `$name! $arg` 的结构时，该结构并不一定是宏，可能是其它语法扩展，比如过程宏。
*   所有宏的输入都是非叶节点的单个标记树。
*   宏（其实所有一般意义上的语法扩展）都将作为抽象语法树的 *一部分* 被解析。

> 此外：由于第一点，下面要谈到的内容（包括下一节）将适用于一般的语法拓展。[^writer-is-lazy]

[^writer-is-lazy]: 打出“宏”比打出“语法拓展”四个字要方便太多 :)

最后一点最为重要，它带来了一些深远的影响。
由于宏将被解析进 AST 中，宏 *只能* 出现在那些明确支持它们出现的位置。
具体来说，宏能在如下位置出现：

*   模式 (pattern) 
*   语句 (statement) 
*   表达式 (expression)
*   条目 (item) （包括 `impl` 块）
*   [类型][type-macros]

一些并不支持的位置包括：

*   标识符 (identifier)
*   `match` 分支
*   结构体的字段中

在上述第一个列表——支持的位置 之外，绝对没有任何地方有使用宏的可能。

[attributes]: https://doc.rust-lang.org/reference/attributes.html
[built-in attributes]: https://doc.rust-lang.org/reference/attributes.html#built-in-attributes-index
[macro attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#attribute-macros
[derive attributes]: https://doc.rust-lang.org/reference/procedural-macros.html#derive-macro-helper-attributes
[procedural macros]: https://doc.rust-lang.org/reference/procedural-macros.html
[`format!`]: https://doc.rust-lang.org/std/macro.format.html
[`format_args!`]: https://doc.rust-lang.org/std/macro.format_args.html
[type-macros]: https://github.com/rust-lang/rfcs/blob/161ce8a26e70226a88e0d4d43c7914a714050330/text/0873-type-macros.md
