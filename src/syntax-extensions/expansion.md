# 展开

展开相对简单。在生成 AST **之后**，和编译器对程序进行语义理解之前，编译器将会对所有语法拓展进行展开。

这一过程包括：遍历 AST，确定所有语法拓展调用的位置，并把它们替换成展开的内容。

每当编译器遇见一个语法扩展，都会根据上下文解析成有限语法元素集中的一个。

举例来说，如果在模块作用域内调用语法拓展，那么编译器就将它的展开结果解析为表示某项条目 (item) 的 AST
节点；如果在表达式的位置上调用语法拓展，那么编译器就将它的展开结果解析为表示表达式的 AST 节点。

事实上，一个语义扩展的展开结果会变成以下一种情况：

* 一个表达式
* 一个模式
* 一个类型
* 零或多个条目（包括的 `impl` 块）
* 零或多个语句

换句话讲，语法拓展调用所在的位置，决定了该语法拓展展开结果被解读的方式。

编译器会操作 AST 节点，把语法拓展调用处的节点完全替换成输出的节点。这一替换是结构性 (structural)
的，而非织构性 (textural) 的。

比如思考以下代码：

```rust,ignore
let eight = 2 * four!();
```

我们可将这部分 AST 可视化地表示为：

```text
┌─────────────┐
│ Let         │
│ name: eight │   ┌─────────┐
│ init: ◌     │╶─╴│ BinOp   │
└─────────────┘   │ op: Mul │
                ┌╴│ lhs: ◌  │
     ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────────┐
     │ LitInt │╶┘ └─────────┘ └╴│ Macro      │
     │ val: 2 │                 │ name: four │
     └────────┘                 │ body: ()   │
                                └────────────┘
```

根据上下文，`four!()` **必须**展开成一个表达式（initializer
只可能是表达式）。因此，无论实际展开的结果如何，它都将被解读成一个完整的表达式。

此处假设 `four!` 成其展开结果为表达式 `1 + 3`。故而，展开后将 AST 变为：

```text
┌─────────────┐
│ Let         │
│ name: eight │   ┌─────────┐
│ init: ◌     │╶─╴│ BinOp   │
└─────────────┘   │ op: Mul │
                ┌╴│ lhs: ◌  │
     ┌────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
     │ LitInt │╶┘ └─────────┘ └╴│ BinOp   │
     │ val: 2 │                 │ op: Add │
     └────────┘               ┌╴│ lhs: ◌  │
                   ┌────────┐ │ │ rhs: ◌  │╶┐ ┌────────┐
                   │ LitInt │╶┘ └─────────┘ └╴│ LitInt │
                   │ val: 1 │                 │ val: 3 │
                   └────────┘                 └────────┘
```

这又能被重写成：

```rust,ignore
let eight = 2 * (1 + 3);
```

注意，虽然表达式本身不包含括号，但这里仍然加上了它们。这是因为，编译器总是将语法拓展的展开结果看作完整的
AST 节点，而**不是**仅仅把它视为一列标记。

换句话说，即便不显式地把复杂的表达式用括号包起来，编译器也不可能“错意”语法拓展替换的结果，或者改变求值顺序。

语法拓展被当作 AST 节点展开，这一观点非常重要，它造成两大影响：

* 语法拓展不仅调用位置有限制，其展开结果也只能跟语法解析器在该位置所预期的 AST 节点种类一致。
* 因此，语法拓展**必定无法**展开成不完整或不合语法的结构。

有关展开还有一点值得注意：如果某语法扩展的展开结果包含**另一个**语法扩展调用，那会怎么样？

例如，上述 `four!` 如果被展开成了 `1 + three!()`，会发生什么?

```rust,ignore
let x = four!();
```

展开成：

```rust,ignore
let x = 1 + three!();
```

编译器将会检查扩展结果中是否包含更多的语法拓展调用；如果有，它们将被进一步展开。

因此，上述 AST 节点将被再次展开成：

```rust,ignore
let x = 1 + 3;
```

这里的要点是，语法拓展展开发生在“传递”过程中；要完全展开所有调用，就需要同样多的传递。

嗯，也不全是如此。事实上，编译器为此设置了一个上限。它被称作语法拓展的递归上限，默认值为 128。如果第
128 次展开结果仍然包含语法拓展调用，编译器将会终止并返回一个递归上限溢出的错误信息。

此上限可通过 [`#![recursion_limit="…"]`][recursion_limit] 来提高，但这种改写必须是 crate
级别的。 一般来讲，可能的话最好还是尽量让语法拓展展开递归次数保持在默认值以下，因为会影响编译时间。

[recursion_limit]: https://doc.rust-lang.org/reference/attributes/limits.html#the-recursion_limit-attribute