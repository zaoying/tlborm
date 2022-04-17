# 片段分类符

正如在 [`macro_rules`](../macro_rules.md) 一章看到的，
1.52 版本的 Rust 已有 13 个片段分类符 (Fragment Specifiers，以下简称分类符)[^metavariables] 。
这一节会更深入地探讨他们之中的细节，每次都会展示几个匹配的例子。

> 注意：除了 `ident`、`lifetime` 和 `tt` 分类符之外，
> 其余的分类符在匹配后生成的 AST 是不清楚的 (opaque)，
> 这使得在之后的宏调用时不可能检查 (inspect) 捕获的结果。

* [`item`](#item)
* [`block`](#block)
* [`stmt`](#stmt)
* [`pat`](#pat)
* [`expr`](#expr)
* [`ty`](#ty)
* [`ident`](#ident)
* [`path`](#path)
* [`tt`](#tt)
* [`meta`](#meta)
* [`lifetime`](#lifetime)
* [`vis`](#vis)
* [`literal`](#literal)

[^metavariables]: 最新内容可参考 *Reference* 的 [Metavariables](https://doc.rust-lang.org/nightly/reference/macros-by-example.html#metavariables) 一节。

## `item`

`item` 分类符只匹配 Rust 的 [item](https://doc.rust-lang.org/reference/items.html) 
的 **定义** (definitions) ，
不会匹配指向 item 的标识符 (identifiers)。例子：

```rust,editable
macro_rules! items {
    ($($item:item)*) => ();
}

items! {
    struct Foo;
    enum Bar {
        Baz
    }
    impl Foo {}
    /*...*/
}
# fn main() {}
```

[`item`](https://doc.rust-lang.org/reference/items.html) 
是在编译时完全确定的，通常在程序执行期间保持固定，并且可以驻留在只读存储器中。具体指：

*   [modules](https://doc.rust-lang.org/reference/items/modules.html)
*   [`extern crate` declarations](https://doc.rust-lang.org/reference/items/extern-crates.html)
*   [`use` declarations](https://doc.rust-lang.org/reference/items/use-declarations.html)
*   [function definitions](https://doc.rust-lang.org/reference/items/functions.html)
*   [type definitions](https://doc.rust-lang.org/reference/items/type-aliases.html)
*   [struct definitions](https://doc.rust-lang.org/reference/items/structs.html)
*   [enumeration definitions](https://doc.rust-lang.org/reference/items/enumerations.html)
*   [union definitions](https://doc.rust-lang.org/reference/items/unions.html)
*   [constant items](https://doc.rust-lang.org/reference/items/constant-items.html)
*   [static items](https://doc.rust-lang.org/reference/items/static-items.html)
*   [trait definitions](https://doc.rust-lang.org/reference/items/traits.html)
*   [implementations](https://doc.rust-lang.org/reference/items/implementations.html)
*   [`extern` blocks](https://doc.rust-lang.org/reference/items/external-blocks.html)

## `block`

`block` 分类符只匹配 [block 表达式](https://doc.rust-lang.org/reference/expressions/block-expr.html) 。

块 (block) 由 `{` 开始，接着是一些语句，最后是可选的表达式，然后以 `}` 结束。
块的类型要么是最后的值表达式类型，要么是 `()` 类型。

```rust,editable
macro_rules! blocks {
    ($($block:block)*) => ();
}

blocks! {
    {}
    {
        let zig;
    }
    { 2 }
}
# fn main() {}
```

## `stmt`

`stmt` 分类符只匹配的 语句 ([statement](https://doc.rust-lang.org/reference/statements.html))。
除非 item 语句要求结尾有分号，否则 **不会** 匹配语句最后的分号。

什么叫 item 语句要求结尾有分号呢？单元结构体 (Unit-Struct) 就是一个简单的例子，
因为它定义中必须带上结尾的分号。

赶紧用例子展示上面说的是啥意思吧。下面的宏只给出它所捕获的内容，因为有几行不能通过编译。

```rust,editable
macro_rules! statements {
    ($($stmt:stmt)*) => ($($stmt)*);
}

fn main() {
    statements! {
        struct Foo;
        fn foo() {}
        let zig = 3
        let zig = 3;
        3
        3;
        if true {} else {}
        {}
    }
}
```

你可以根据报错内容试着删除不能编译的代码，结合 [`stmt`](#stmt) 小节开头的文字再琢磨琢磨。
你如果正浏览使用 mdbook 渲染的页面，那么可以直接运行和修改这段代码。

虽然源代码编译失败，但是我们可以展开宏[^debugging]，
使用 [playground](https://play.rust-lang.org/) 的
`Expand macros` 工具 (tool)；或者把代码复制到本地，在 nightly Rust 版本中使用
`cargo rustc -- -Zunstable-options --pretty=expanded` 命令得到宏展开结果：

```rust,ignore
# warning: unnecessary trailing semicolon
#   --> src/main.rs:10:20
#    |
# 10 |         let zig = 3;
#    |                    ^ help: remove this semicolon
#    |
#    = note: `#[warn(redundant_semicolons)]` on by default
# 
# warning: unnecessary trailing semicolon
#   --> src/main.rs:12:10
#    |
# 12 |         3;
#    |          ^ help: remove this semicolon
# 
# #![feature(prelude_import)]
# #[prelude_import]
# use std::prelude::rust_2018::*;
# #[macro_use]
# extern crate std;
# macro_rules! statements { ($ ($ stmt : stmt) *) => ($ ($ stmt) *) ; }

fn main() {
    struct Foo;
    fn foo() { }
    let zig = 3;
    let zig = 3;
    ;
    3;
    3;
    ;
    if true { } else { }
    { }
}
```

由此我们知道：

1. 虽然 `stmt` 分类符没有捕获语句末尾的分号，但它依然在所需的时候返回了 (emit) 语句。
原因很简单，分号本身就是有效的语句。
所以我们实际输入 11 个语句调用了宏，而不是 8 个！

2. 在这里你应该注意到：`struct Foo;` 被匹配到了。
否则我们会看到像其他情况一样有一个额外 `;` 语句。
由前所述，这能想通：item 语句需要分号，所以这个分号能被匹配到。

3. 仅由块表达式或控制流表达式组成的表达式结尾没有分号，
其余的表达式捕获后产生的表达式会尾随一个分号（在这个例子中，正是这里出错）。

这里提到的细节能在 Reference 的 [statement](https://doc.rust-lang.org/reference/statements.html)
一节中找到。

[^debugging]: 可阅读 [调试](./debugging.html) 一章

## `pat`

`pat` 分类符用于匹配任何形式的模式 ([pattern](https://doc.rust-lang.org/reference/patterns.html))。

```rust,editable
macro_rules! patterns {
    ($($pat:pat)*) => ();
}

patterns! {
    "literal"
    _
    0..5
    ref mut PatternsAreNice
}
# fn main() {}
```

## `expr`

`expr` 分类符用于匹配任何形式的表达式
([expression](https://doc.rust-lang.org/reference/expressions.html))。

（如果把 Rust 视为面向表达式的语言，那么它有很多种表达式。）

```rust,editable
macro_rules! expressions {
    ($($expr:expr)*) => ();
}

expressions! {
    "literal"
    funcall()
    future.await
    break 'foo bar
}
# fn main() {}
```

## `ty`

`ty` 分类符用于匹配任何形式的类型表达式 ([type expression](https://doc.rust-lang.org/reference/types.html#type-expressions))。

类型表达式是在 Rust 中指代类型的语法。

```rust,editable
macro_rules! types {
    ($($type:ty)*) => ();
}

types! {
    foo::bar
    bool
    [u8]
}
# fn main() {}
```

## `ident`

`ident` 分类符用于匹配任何形式的标识符或者关键字。
([identifier](https://doc.rust-lang.org/reference/identifiers.html))。

```rust,editable
macro_rules! idents {
    ($($ident:ident)*) => ();
}

idents! {
    // _ /* `_` 不是标识符，而是一种模式 */
    foo
    async
    O_________O
    _____O_____
}
# fn main() {}
```

## `path`

`path` 分类符用于匹配类型中的路径
([TypePath](https://doc.rust-lang.org/reference/paths.html#paths-in-types)) 。

```rust,editable
macro_rules! paths {
    ($($path:path)*) => ();
}

paths! {
    ASimplePath
    ::A::B::C::D
    G::<eneri>::C
}
# fn main() {}
```

## `tt`

`tt` 分类符用于匹配标记树 (TokenTree)。
如果你是新手，对标记树不了解，那么需要回顾本书 
[标记树](../syntax/source-analysys.html#标记树-token-trees)
一节。`tt` 分类符是最有作用的分类符之一，因为它能匹配几乎所有东西，
而且能够让你在使用宏之后检查 (inspect) 匹配的内容。

## `meta`

`meta` 分类符用于匹配属性 ([attribute](https://doc.rust-lang.org/reference/attributes.html))，
准确地说是属性里面的内容。通常你会在 `#[$meta:meta]` 或 `#![$meta:meta]` 模式匹配中
看到这个分类符。

```rust,editable
macro_rules! metas {
    ($($meta:meta)*) => ();
}

metas! {
    ASimplePath
    super::man
    path = "home"
    foo(bar)
}
# fn main() {}
```

> 针对文档注释简单说一句：
> 文档注释其实是具有 `#[doc="…"]` 形式的属性，`...` 实际上就是注释字符串，
> 这意味着你可以在在宏里面操作文档注释！

## `lifetime`

`lifetime` 分类符用于匹配生命周期注解或者标签
([lifetime or label](https://doc.rust-lang.org/reference/tokens.html#lifetimes-and-loop-labels))。
它与 [`ident`](#ident) 很像，但是 `lifetime` 会匹配到前缀 `''` 。

```rust,editable
macro_rules! lifetimes {
    ($($lifetime:lifetime)*) => ();
}

lifetimes! {
    'static
    'shiv
    '_
}
# fn main() {}
```

## `vis`

`vis` 分类符会匹配 **可能为空** 的内容。
([Visibility qualifier](https://doc.rust-lang.org/reference/visibility-and-privacy.html))。
重点在于“可能为空”。你可能想到这是隐藏了 `?` 重复操作符的分类符，
这样你就不用直接在反复匹配时使用 `?` —— 其实你不能将它和 `?` 一起在重复模式匹配中使用。

```rust,editable
macro_rules! visibilities {
    // 注意这个逗号，`vis` 分类符自身不会匹配到逗号
    ($($vis:vis,)*) => ();
}

visibilities! {
    ,
    pub,
    pub(crate),
    pub(in super),
    pub(in some_path),
}
# fn main() {}
```

`vis` 实际上只支持例子里的几种方式，因为这里的 visibility 指的是可见性，与私有性相对。
而涉及这方面的内容只有与 `pub` 的关键字。所以，`vis` 在关心匹配输入的内容是公有还是私有时有用。

## `literal`

`literal` 分类符用于匹配字面表达式 
([literal expression](https://doc.rust-lang.org/reference/expressions/literal-expr.html))。

```rust,editable
macro_rules! literals {
    ($($literal:literal)*) => ();
}

literals! {
    -1
    "hello world"
    2.3
    b'b'
    true
}
# fn main() {}
```
