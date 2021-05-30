# 调试

调试需要用到 Rust 的 nightly 版本。

## `trace_macros!`

`rustc` 提供了一些调试 `macro_rules!` 宏的工具。其中最有用的就是 [`trace_macros!`]。

给 [`trace_macros!`] 传入 `true`，从而给编译器发出指令，让这开始的 `macro_rules!` 宏在展开前被调用。

给 [`trace_macros!`] 传入 `false`，关闭追踪功能。

请看这个例子：

```rust,editable
# // 注意：必须使用 nightly 版本的 Rust 编译这段代码
#![feature(trace_macros)]

macro_rules! each_tt {
    () => {};
    ($_tt:tt $($rest:tt)*) => {each_tt!($($rest)*);};
}

each_tt!(foo bar baz quux);
trace_macros!(true);
each_tt!(spim wak plee whum);
trace_macros!(false);
each_tt!(trom qlip winp xod);
# 
# fn main() {}
```

在第二条宏的前后开启和关闭调试，从而得到以下打印结果：

```text
note: trace_macro
  --> src/main.rs:11:1
   |
11 | each_tt!(spim wak plee whum);
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: expanding `each_tt! { spim wak plee whum }`
   = note: to `each_tt ! (wak plee whum) ;`
   = note: expanding `each_tt! { wak plee whum }`
   = note: to `each_tt ! (plee whum) ;`
   = note: expanding `each_tt! { plee whum }`
   = note: to `each_tt ! (whum) ;`
   = note: expanding `each_tt! { whum }`
   = note: to `each_tt ! () ;`
   = note: expanding `each_tt! {  }`
   = note: to ``
```

它在调试递归很深的宏时尤其有用。

此外，你可以在命令行里，给编译指令附加 `-Z trace-macros` 来打印追踪的宏。

`trace_macros!(false);` 之后的宏不会被这个附加指令追踪到，所以这里会追踪前两个宏。

参考命令：`cargo rustc --bin binary_name  -- -Z trace-macros`

## `log_syntax!`

另一有用的宏是`log_syntax!`。
它将使得编译器输出所有经过编译器处理的标记 (tokens)。

举个例子，下述代码可以让编译器唱一首歌：

```rust,editable
# // 注意：必须使用 nightly 版本的 Rust 编译这段代码
#![feature(log_syntax)]

macro_rules! sing {
    () => {};
    ($tt:tt $($rest:tt)*) => {log_syntax!($tt); sing!($($rest)*);};
}

sing! {
    ^ < @ < . @ *
    '\x08' '{' '"' _ # ' '
    - @ '$' && / _ %
    ! ( '\t' @ | = >
    ; '\x08' '\'' + '$' ? '\x7f'
    , # '"' ~ | ) '\x07'
}
# 
# fn main() {}
```
比起 `trace_macros!` 来说，它能够做一些更有针对性的调试。
打印结果：
```text
^
<
@
<
.
@
*
'\x08'
'{'
'"'
_
#
' '
-
@
'$'
&&
/
_
%
!
('\t' @ | = > ; '\x08' '\'' + '$' ? '\x7f', # '"' ~ |)
'\x07'
```

## `--pretty=expanded`

有时问题出在宏展开后的结果里。
对于这种情况，可给 rustc 编译命令添加 `--pretty` 参数来勘察。
下面给出代码：

```rust,ignore
// 给 `String` 初始化函数起一个简写
macro_rules! S {
    ($e:expr) => {String::from($e)};
}

fn main() {
    let world = S!("World");
    println!("Hello, {}!", world);
}
```

使用下面一种方式编译这段名为 `hello.rs` 的脚本文件：

```shell
# 对单个 rs 文件
rustc -Z unstable-options --pretty expanded hello.rs
# 对项目里的二进制 rs 文件
cargo rustc --bin hello -- -Z unstable-options --pretty=expanded
```

将输出如下内容（略经排版）：

```rust,ignore
#![feature(prelude_import)]
#[prelude_import]
use std::prelude::rust_2018::*;
#[macro_use]
extern crate std;
macro_rules! S {
    ($e:expr) => {
        String::from($e)
    };
}
fn main() {
    let world = String::from("World");
    {
        ::std::io::_print(
            ::core::fmt::Arguments::new_v1(
            &["Hello, ", "!\n"],
            &match (&world,) {
                (arg0,) => {
                    [::core::fmt::ArgumentV1::new(arg0, ::core::fmt::Display::fmt)]
                }
            })
        );
    };
}
```

`--pretty` 还有其它一些可用选项，可通过 `rustc -Z unstable-options --help -v` 来列出。
此处并不提供该选项表；因为，正如指令本身所暗示的，列表中的一切内容在任何时间点都有可能发生改变。

## 其他调试工具

1. `rustc` 提供了帮助调试的工具：不仅仅是上面谈到的 `--pretty=expanded` 命令行参数选项，
[dtolnay] 编写的 [`cargo-expand`] 插件对 `rustc` 调试功能进行基本的封装[^example]。

2. 你还可以使用 [playground] ：点击页面右上方的 `TOOLS` 按钮，再点击宏展开按钮 (expand macros)。

3. 另一个很棒的工具是 [lukaslueg] 编写的 [`macro_railroad`] lib，
它能可视化地生成 Rust `macro_rules!` 宏的语法图 (syntax diagrams)。


[`trace_macros!`]:https://doc.rust-lang.org/std/macro.trace_macros.html
[`log_syntax!`]:https://doc.rust-lang.org/std/macro.log_syntax.html
[`cargo-expand`]:https://github.com/dtolnay/cargo-expand
[dtolnay]:https://github.com/dtolnay
[playground]:https://play.rust-lang.org/
[lukaslueg]:https://github.com/lukaslueg
[`macro_railroad`]:https://github.com/lukaslueg/macro_railroad

[^example]: 例子见 [计数-bit-twiddling](../building-blocks/counting.html#bit-twiddling)
