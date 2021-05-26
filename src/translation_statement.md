# 翻译说明

本书是续写版本，续写的版本由 Veykril 撰稿，续作对原作有补充和删改。

- 原作：[repo](https://github.com/DanielKeep/tlborm) | [渲染版](https://danielkeep.github.io/tlborm/)
- 原作中文翻译：[repo](https://github.com/DaseinPhaos/tlborm) | [渲染版](https://www.bookstack.cn/read/DaseinPhaos-tlborm-chinese/README.md)

原作及其翻译渲染版本没有使用 mdbook 构建，而是使用 py 来生成 HTML。
在发布文档和运行样板代码方面诸多不便。

而且由于原作在 2016 年没再更新，其内容基于 Rust 2015 版本，
续写的版本也只是把过时的细节更新至 2018 版本。

今年即将发布 2021 版本，**或许** 本书的内容会继续修正吧。
毕竟这本书在阐述声明宏方面搭建了一个很小巧精美的骨架。

续作及本翻译渲染版本使用 mdbook 构建：
- 续作：[repo](https://github.com/veykril/tlborm) | [渲染版](https://veykril.github.io/tlborm/)
- 续作中文翻译：[repo](https://github.com/zjp-CN/tlborm) | [渲染版](https://zjp-cn.github.io/tlborm) | [加速查看站点](http://129.28.186.100/tlborm)

主要补充的部分在于：

- *macro_rules!* ：[Metavariables](./macros/macro_rules.html#metavariables)
- [片段分类符](./macros/minutiae/fragment-specifiers.html)
- 调试：[其他调试工具](./macros/minutiae/debugging.html#其他调试工具)
- [作用域](./macros/minutiae/scoping.md)
- [导入/导出宏](./macros/minutiae/import-export.md)


此外，所有 [`code`](#) 蓝色样式、光标移上去有下划线的内容（普通正文或者行内代码）都是链接，可以跳转。
无链接的行内代码样式是这样的：`code` 。
