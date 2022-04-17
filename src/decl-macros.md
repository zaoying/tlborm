# 方法论

本章将介绍 Rust 的声明宏系统： [`macro_rules!`][mbe]，这在 [Reference: Macro-By-Example][mbe] 一节中介绍过。
与其根据实际例子来阐述，不如试着对宏系统的工作原理进行完整而透彻的解释。
因此，这章意在解释宏系统如何作为一个整体运作，而不是给想要一步步引导编写宏的人准备的。

此外：Rust Book 的 [macro 一节][Macros chapter of the Rust Book] 更浅显易懂、站在更高的角度做出了解释；
参考手册的 [macros-by-example](https://doc.rust-lang.org/reference/macros-by-example.html) 一节，
和本书 [实战](./macros-practical.html) 一章都对单个宏的使用做出了示范。

[mbe]: https://doc.rust-lang.org/reference/macros-by-example.html
[Macros chapter of the Rust Book]: https://doc.rust-lang.org/book/ch19-06-macros.html
