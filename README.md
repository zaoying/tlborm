# The Little Book of Rust Macros （Rust 宏之书）

> 注意：这是对 [Daniel Keep 撰写的书](https://github.com/DanielKeep/tlborm) 的续写，
> 自 2016 年初夏以来，那本书就一直没再更新。

本书使用 [mdBook](https://github.com/rust-lang/mdBook) 工具生成。
您可以浏览本书的 [英文版本](https://veykril.github.io/tlborm/)，
和 [github 仓库](https://github.com/veykril/tlborm)。（译者注：续写的版本由 Veykril 编写 ）





This book is an attempt to distill the Rust community's collective knowledge of Rust macros,
`Macros by Example` to be precise.  As such, both additions (in the form of pull requests) and
requests (in the form of issues) are welcome.
The [original Little Book of Rust Macros](https://github.com/DanielKeep/tlborm) has helped me
immensely with understanding ***Macros by Example*** style macros while I was still learning the
language. Unfortunately, the author of the book has since left the project untouched, while the Rust
language as well as it's macro-system keeps evolving. Which is why I wanted to revive the project
and keep it up to date with current Rust helping other newcomers understand macros, a part of the
language a lot of people seem to have trouble with.

> This book expects you to have basic knowledge of Rust, it will not explain language features or
> constructs that are irrelevant to macros. No prior knowledge of macros is assumed. Having read and
> understood the first seven chapters of the [Rust Book](https://doc.rust-lang.org/stable/book/) is
> a must, though having read the majority of the book is recommended.

## Thanks

Thanks to Daniel Keep for the original work and thanks to the following people for suggestions and
corrections to the original: IcyFoxy, Rym, TheMicroWorm, Yurume, akavel, cmr, eddyb, ogham, and
snake_case.

## License

This work inherits the licenses of the original, hence it is licensed under both the
[Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)
and the [MIT license](http://opensource.org/licenses/MIT).
