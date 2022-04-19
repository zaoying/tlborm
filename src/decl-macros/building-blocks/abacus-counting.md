# 算盘计数

## 描述分析

> 临时信息：需要更合适的例子。
该用例采用 Rust 分组机制无法表示的匹配嵌套结构，
实在是过于特殊，因此不适作为例子使用。

> 注意：此节假设读者已经了解 [下推累积](../patterns/push-down-acc.md) 
以及 [标记树撕咬机](../patterns/tt-muncher.md) 。

```rust,editable
macro_rules! abacus {
    ((- $($moves:tt)*) -> (+ $($count:tt)*)) => {
        abacus!(($($moves)*) -> ($($count)*))
    };
    ((- $($moves:tt)*) -> ($($count:tt)*)) => {
        abacus!(($($moves)*) -> (- $($count)*))
    };
    ((+ $($moves:tt)*) -> (- $($count:tt)*)) => {
        abacus!(($($moves)*) -> ($($count)*))
    };
    ((+ $($moves:tt)*) -> ($($count:tt)*)) => {
        abacus!(($($moves)*) -> (+ $($count)*))
    };

    // Check if the final result is zero.
    (() -> ()) => { true };
    (() -> ($($count:tt)+)) => { false };
}

fn main() {
    let equals_zero = abacus!((++-+-+++--++---++----+) -> ());
    assert_eq!(equals_zero, true);
}
```

这个例子所用的技巧用在如下情况：
记录的计数会发生变化，且初始值为零或在零附近，且必须支持如下操作：

* 增加一；
* 减少一；
* 与 0 （或任何其它固定的有限值）相比较；

数值 n 将由一组共 n 个相同的特定标记来表示。
对数值的修改操作将采用 [下推累积](../patterns/push-down-acc.md) 模式由递归调用完成。
假设所采用的特定标记是 `x` ，则上述操作可实现为：

* 增加一：匹配`($($count:tt)*)`并替换为`(x $($count)*)`。
* 减少一：匹配`(x $($count:tt)*)`并替换为`($($count)*)`。
* 与0相比较：匹配`()`。
* 与1相比较：匹配`(x)`。
* 与2相比较：匹配`(x x)`。
* *(依此类推...)*

作用于计数值的操作将所选的标记来回摆动，如同算盘摆动算子。[^abacus]


[^abacus]: 在这句极度单薄的辩解下，隐藏着选用此名称的 *真实* 理由：
避免造出又一个名含“标记”的术语。今天就该跟你认识的作者谈谈避免
[语义饱和](https://en.wikipedia.org/wiki/Semantic_satiation) 吧！
公平来讲，本来也可以称它为
[“一元计数(unary counting)”](https://en.wikipedia.org/wiki/Unary_numeral_system) 。

在想表示负数的情况下，值 *-n* 可被表示成 *n* 个相同的其它标记。
在上例中，值 *+n* 被表示成 *n* 个 `+` 标记，而值 *-m* 被表示成 *m* 个 `-` 标记。

有负数的情况下操作起来稍微复杂一些，
增减操作在当前数值为负时实际上互换了角色。
给定 `+` 和 `-` 分别作为正数与负数标记，相应操作的实现将变成：

* 增加一：
  * 匹配 `()` 并替换为 `(+)` 
  * 匹配 `(- $($count:tt)*)` 并替换为 `($($count)*)`
  * 匹配 `($($count:tt)+)` 并替换为 `(+ $($count)+)`
* 减少一：
  * 匹配 `()` 并替换为 `(-)`
  * 匹配 `(+ $($count:tt)*)` 并替换为 `($($count)*)`
  * 匹配 `($($count:tt)+)` 并替换为 `(- $($count)+)`
* 与 0 相比较：匹配 `()`
* 与 +1 相比较：匹配 `(+)`
* 与 -1 相比较：匹配 `(-)`
* 与 +2 相比较：匹配 `(++)`
* 与 -2 相比较：匹配 `(--)`
* *(依此类推...)*

注意在顶部的示例中，某些规则被合并到一起了
（举例来说，对 `()` 及 `($($count:tt)+)` 的增加操作被合并为对
`($($count:tt)*)` 的增加操作）。

如果想要提取出所计数目的实际值，可再使用普通的 
[计数宏](../building-blocks/counting.md) 。对上例来说，终结规则可换为：

```rust,ignore
macro_rules! abacus {
    // ...

    // 下列规则将计数替换成实际值的表达式
    (() -> ()) => {0};
    (() -> (- $($count:tt)*)) => {
        - ( count_tts!($( $count_tts:tt )*) )
    };
    (() -> (+ $($count:tt)*)) => {
        count_tts!($( $count_tts:tt )*)
    };
}

// 计数一章任选一个宏
macro_rules! count_tts {
    // ...
}
```

> <abbr title="Just for this example">仅限此例</abbr>：
严格来说，想要达到此例的效果，没必要做的这么复杂。
如果你不需要在宏中匹配所计的值，可直接采用重复来更加高效地实现：
>
> ```RUST,ignore
> macro_rules! abacus {
>     (-) => {-1};
>     (+) => {1};
>     ($( $moves:tt )*) => {
>         0 $(+ abacus!($moves))*
>     }
> }
> ```


## 算盘游戏

> 译者注：这章原作者的表述实在过于啰嗦，但是这个例子的确很有意思。
基于这个例子框架，我给出如下浅显而完整的样例代码（可编辑运行）：
```rust,editable
macro_rules! abacus {
    ((- $($moves:tt)*) -> (+ $($count:tt)*)) => {
        {
            println!("{} [-]{} | [+]{}", "-+1", stringify!($($moves)*), stringify!($($count)*));
            abacus!(($($moves)*) -> ($($count)*))
        }
    };
    ((- $($moves:tt)*) -> ($($count:tt)*)) => {
        {
            println!("{} [-]{} | - {}", "- 2", stringify!($($moves)*), stringify!($($count)*));
            abacus!(($($moves)*) -> (- $($count)*))
        }
    };
    ((+ $($moves:tt)*) -> (- $($count:tt)*)) => {
        {
            println!("{} [+]{} | [-]{}", "+-3", stringify!($($moves)*), stringify!($($count)*));
            abacus!(($($moves)*) -> ($($count)*))
        }
    };
    ((+ $($moves:tt)*) -> ($($count:tt)*)) => {
        {
            println!("{} [+]{} | + {}", "+ 4", stringify!($($moves)*), stringify!($($count)*));
            abacus!(($($moves)*) -> (+ $($count)*))
        }
    };

    (() -> ()) => {0};
    (() -> (- $($count:tt)*)) => {{-1 + abacus!(() -> ($($count)*)) }};
    (() -> (+ $($count:tt)*)) => {{1 + abacus!(() -> ($($count)*)) }};
}

fn main() {
    println!("算盘游戏：左边与右边异号时抵消；非异号时，把左边的符号转移到右边；左边无符号时，游戏结束，计算右边得分");
    println!("图示注解：左右符号消耗情况，分支编号，[消失的符号] 左边情况 | [消失的符号] 右边情况\n");

    println!("计数结果：{}\n", abacus!((++-+-+) -> (--+-+-)));
    println!("计数结果：{}\n", abacus!((++-+-+) -> (++-+-+)));
    println!("计数结果：{}\n", abacus!((---+) -> ()));
    println!("计数结果：{}\n", abacus!((++-+-+) -> ()));
    println!("计数结果：{}\n", abacus!((++-+-+++--++---++----+) -> ())); // 这是作者给的例子 :)
}
```

打印结果：

```text
算盘游戏：左边与右边异号时抵消；非异号时，把左边的符号转移到右边；左边无符号时，游戏结束，计算右边得分
图示注解：左右符号消耗情况，分支编号，[消失的符号] 左边情况 | [消失的符号] 右边情况

+-3 [+]+ - + - + | [-]- + - + -
+-3 [+]- + - + | [-]+ - + -
-+1 [-]+ - + | [+]- + -
+-3 [+]- + | [-]+ -
-+1 [-]+ | [+]-
+-3 [+] | [-]
计数结果：0

+ 4 [+]+ - + - + | + + + - + - +
+ 4 [+]- + - + | + + + + - + - +
-+1 [-]+ - + | [+]+ + + - + - +
+ 4 [+]- + | + + + + - + - +
-+1 [-]+ | [+]+ + + - + - +
+ 4 [+] | + + + + - + - +
计数结果：4

- 2 [-]- - + | - 
- 2 [-]- + | - -
- 2 [-]+ | - - -
+-3 [+] | [-]- -
计数结果：-2

+ 4 [+]+ - + - + | + 
+ 4 [+]- + - + | + +
-+1 [-]+ - + | [+]+
+ 4 [+]- + | + +
-+1 [-]+ | [+]+
+ 4 [+] | + +
计数结果：2

+ 4 [+]+ - + - + + + - - + + - - - + + - - - - + | + 
+ 4 [+]- + - + + + - - + + - - - + + - - - - + | + +
-+1 [-]+ - + + + - - + + - - - + + - - - - + | [+]+
+ 4 [+]- + + + - - + + - - - + + - - - - + | + +
-+1 [-]+ + + - - + + - - - + + - - - - + | [+]+
+ 4 [+]+ + - - + + - - - + + - - - - + | + +
+ 4 [+]+ - - + + - - - + + - - - - + | + + +
+ 4 [+]- - + + - - - + + - - - - + | + + + +
-+1 [-]- + + - - - + + - - - - + | [+]+ + +
-+1 [-]+ + - - - + + - - - - + | [+]+ +
+ 4 [+]+ - - - + + - - - - + | + + +
+ 4 [+]- - - + + - - - - + | + + + +
-+1 [-]- - + + - - - - + | [+]+ + +
-+1 [-]- + + - - - - + | [+]+ +
-+1 [-]+ + - - - - + | [+]+
+ 4 [+]+ - - - - + | + +
+ 4 [+]- - - - + | + + +
-+1 [-]- - - + | [+]+ +
-+1 [-]- - + | [+]+
-+1 [-]- + | [+]
- 2 [-]+ | - 
+-3 [+] | [-]
计数结果：0
```
