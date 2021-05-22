# 语法拓展
在说起 `macro_rules!` （声明宏）之前，值得讨论其基于的一般机制：语法拓展。
为了解释这个机制，我们必须探讨 Rust 源代码

Before talking about `macro_rules!`, it is worthwhile to discuss the general mechanism they are built on:
syntax extensions. To do that, we must discuss how Rust source is processed by the compiler, and the
general mechanisms on which user-defined `macro_rules!` macros are built.
