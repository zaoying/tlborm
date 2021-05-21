# 方法论

本章将介绍 Rust 的声明宏系统： [`macro_rules!`][mbe]，这在 [Reference: Macro-By-Example][mbe] 一节中介绍过。
与其根据实际例子来阐述，不如试着对宏系统的工作原理进行完整而透彻的解释。
因此，这章意在解释宏系统如何作为一个整体运作，而不是给想要一步步引导编写宏的人准备的。



There is also the [Macros chapter of the Rust Book] which is a more approachable, high-level
explanation, the reference [chapter](https://doc.rust-lang.org/reference/macros-by-example.html)
and the [practical introduction](./macros-practical.html) chapter of this book, which is a guided
implementation of a single macro.


[mbe]: https://doc.rust-lang.org/reference/macros-by-example.html
[Macros chapter of the Rust Book]: https://doc.rust-lang.org/book/ch19-06-macros.html
