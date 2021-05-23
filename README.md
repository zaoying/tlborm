# Rust 宏小册

> 注意：这是对 [Daniel Keep 撰写的书](https://github.com/DanielKeep/tlborm) 的续写，
> 自 2016 年初夏以来，那本书就一直没再更新。

本书使用 [mdBook](https://github.com/rust-lang/mdBook) 工具生成。
你可以浏览本书的 [英文版本](https://veykril.github.io/tlborm/)，
和 github [仓库](https://github.com/veykril/tlborm)。

（译者注：续写的版本由 [Veykril](https://github.com/Veykril) 撰稿。
你也可以浏览 [中文版](https://zjp-cn.github.io/tlborm) 
和 [翻译仓库](https://github.com/zjp-CN/tlborm) 。）

这本书尝试提炼出 Rust 社区对 Rust 宏的共识，准确地说，是 *通过例子* 来讲述宏。
因此，欢迎 PR 补充和提 issue。

在我学习 Rust 的时候，*Little Book of Rust Macros* [原作](https://github.com/DanielKeep/tlborm) 
*通过例子* 的方式非常给力地帮助过我理解宏。
很遗憾的是，Rust 语言与宏系统持续改进时，原作者不再更新书籍。
这也是我想重新继续这本书的原因，让书与当前的 Rust 同步，
以帮助新的 Rust 学习者理解宏——这个让很多人困惑的部分。

> 这本书认为你应该对 Rust 有基本的了解，它不会解释 Rust 语言特性或者与宏有关的结构体。
> 如果你之前对宏没有了解，那么你必须至少阅读和理解了 [Rust Book](https://doc.rust-lang.org/stable/book/) 的前七章——建议阅读完那本书大部分内容。

## 致谢

感谢 Daniel Keep 最初写下这本书。

感谢对原书提出建议和更正的读者：IcyFoxy、 Rym、 TheMicroWorm、 Yurume、 akavel、 cmr、 eddyb、 ogham 和 snake_case。

感谢 [DaseinPhaos](https://github.com/DaseinPhaos/tlborm-chinese) 对原作的翻译。也感谢 Jorel Ali 给 mdbook 提供的 [页面 TOC](https://github.com/JorelAli/mdBook-pagetoc) 小功能。

## 版权声明

这本书沿袭了原作的版权声明，因此具有 [CC BY-SA 4.0](http://creativecommons.org/licenses/by-sa/4.0/) 和 [MIT license](http://opensource.org/licenses/MIT) 的双重许可。
