# 增量式 `TT` “撕咬机”

> 译者注：原文标题为 *incremental `TT` muncher* 。

```rust
macro_rules! mixed_rules {
    () => {};
    (trace $name:ident; $($tail:tt)*) => {
        {
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
    (trace $name:ident = $init:expr; $($tail:tt)*) => {
        {
            let $name = $init;
            println!(concat!(stringify!($name), " = {:?}"), $name);
            mixed_rules!($($tail)*);
        }
    };
}
# 
# fn main() {
#     let a = 42;
#     let b = "Ho-dee-oh-di-oh-di-oh!";
#     let c = (false, 2, 'c');
#     mixed_rules!(
#         trace a;
#         trace b;
#         trace c;
#         trace b = "They took her where they put the crazies.";
#         trace b;
#     );
# }
```

此模式可能是 **最强大** 的宏解析技巧。
通过使用它，一些极其复杂的语法都能得到解析。


“标记树撕咬机” (`TT` muncher) 是一种递归宏，
其工作机制有赖于对输入的顺次、逐步处理 (incrementally processing) 。
处理过程的每一步中，它都将匹配并移除（“撕咬”掉）输入头部 (start) 的一列标记 (tokens)，
得到一些中间结果，然后再递归地处理输入剩下的尾部。

名称中含有“标记树”，是因为输入中尚未被处理的部分总是被捕获在 `$($tail:tt)*` 的形式中。
之所以如此，是因为只有通过使用反复匹配 [`tt`] 才能做到 **无损地** (losslessly) 
捕获住提供给宏的输入部分。

标记树撕咬机仅有的限制，也是整个宏系统的局限：

* 你只能匹配 `macro_rules!` 捕获到的字面值和语法结构。
* 你无法匹配不成对的标记组 (unbalanced group) 。

然而，需要把宏递归的局限性纳入考量。
`macro_rules!` 没有做任何形式的尾递归消除或优化。
在写标记树撕咬机时，建议多花些功夫，尽可能地限制递归调用的次数。

以下两种做法帮助你做到限制宏递归：
1. 对于输入的变化，增加额外的匹配规则（而不是采用中间层并使用递归）[^example]；
2. 对输入句法施加限制，以便于记录追踪标准式的反复匹配。

[^example]: 例子见 [计数-递归](../building-blocks/counting.html#递归)

[`tt`]:../macros/minutiae/fragment-specifiers.md#tt
