<!-- toc -->

# 源代码解析方式

## 标识化 (Tokenization)

Rust程序编译过程的第一阶段是 [标记解析][tokenization]。
在这一过程中，源代码将被转换成一系列的标记
（token，即无法被分割的词法单元；在编程语言世界中等价于“单词”）。

Rust包含多种标记，比如：

* 标识符 (identifiers): `foo`, `Bambous`, `self`, `we_can_dance`, `LaCaravane`, …
* 字面值 (literals): `42`, `72u32`, `0_______0`, `1.0e-40`, `"ferris was here"`, …
* 关键字 (keywords): `_`, `fn`, `self`, `match`, `yield`, `macro`, …
* 符号   (symbols): `[`, `:`, `::`, `?`, `~`, `@`[^wither-at], …

等等。

有些地方值得注意：

1. `self` 既是一个标识符又是一个关键词。
几乎在所有情况下它都被视作是一个关键词，但它有可能被视为标识符。
我们稍后会（骂骂咧咧地）提到这种情况。

2. 关键词里列有一些可疑的家伙，比如 `yield` 和 `macro`。
它们在当前的Rust语言中并没有任何含义，但编译器的确会把它们视作关键词进行解析。
这些词语被保留作语言未来扩充时使用。

3. 符号里也列有一些未被当前语言使用的条目。比如 `<-`，这是历史残留：
目前它被移除了Rust语法，但词法分析器仍然没丢掉它。

4. 注意 `::` 被视作一个独立的标记，而非两个连续的 `:` 。
这一规则适用于截至 Rust 1.2 版本的所有的多字符符号标记。
[^two-lexers]

[^wither-at]: `@` 被用在模式中，用来绑定模式非终止的部分到一个名称——但这似乎被大多数人完全地遗忘了。

[^two-lexers]: 严格来说， Rust 1.46 版本中存在两个词法分析器 (lexer)：
[`rustc_lexer`] 只将单个字符作为 标识 (tokens)；
[`rustc_parse`] 里的 [lexer] 把多个字符作为不同的 标识 (tokens)。

作为对比，某些语言的宏系统正扎根于这一阶段。Rust并非如此。举例来说，从效果来看，C/C++的宏就是在这里得到处理的。^其实不是这也正是下列代码能够运行的原因:

As a point of comparison, it is at *this* stage that some languages have their macro layer, though
Rust does *not*. For example, C/C++ macros are *effectively* processed at this point.
[^lies-damn-lies-cpp] This is why the following code works:
[^cpp-it-seemed-like-a-good-idea-at-the-time]

```c
#define SUB void
#define BEGIN {
#define END }

SUB main() BEGIN
    printf("Oh, the horror!\n");
END
```

[^lies-damn-lies-cpp]: In fact, the C preprocessor uses a different lexical structure to C itself,
but the distinction is *broadly* irrelevant.

[^cpp-it-seemed-like-a-good-idea-at-the-time]: *Whether* it should work is an entirely *different*
question.

### Parsing

The next stage is parsing, where the stream of tokens is turned into an [Abstract Syntax Tree] (AST).
This involves building up the syntactic structure of the program in memory. For example, the token
sequence `1 + 2` is transformed into the equivalent of:

```text
┌─────────┐   ┌─────────┐
│ BinOp   │ ┌╴│ LitInt  │
│ op: Add │ │ │ val: 1  │
│ lhs: ◌  │╶┘ └─────────┘
│ rhs: ◌  │╶┐ ┌─────────┐
└─────────┘ └╴│ LitInt  │
              │ val: 2  │
              └─────────┘
```

The AST contains the structure of the *entire* program, though it is based on purely *lexical*
information. For example, although the compiler may know that a particular expression is referring
to a variable called `a`, at this stage, it has *no way* of knowing what `a` is, or even *where* it
comes from.

It is *after* the AST has been constructed that macros are processed. However, before we can discuss
that, we have to talk about token trees.

## Token trees

Token trees are somewhere between tokens and the AST. Firstly, *almost* all tokens are also token
trees; more specifically, they are *leaves*. There is one other kind of thing that can be a token
tree leaf, but we will come back to that later.

The only basic tokens that are *not* leaves are the "grouping" tokens: `(...)`, `[...]`, and `{...}`.
These three are the *interior nodes* of token trees, and what give them their structure. To give a
concrete example, this sequence of tokens:

```text
a + b + (c + d[0]) + e
```

would be parsed into the following token trees:

```text
«a» «+» «b» «+» «(   )» «+» «e»
          ╭────────┴──────────╮
           «c» «+» «d» «[   ]»
                        ╭─┴─╮
                         «0»
```

Note that this has *no relationship* to the AST the expression would produce; instead of a single
root node, there are *seven* token trees at the root level. For reference, the AST would be:

```text
              ┌─────────┐
              │ BinOp   │
              │ op: Add │
            ┌╴│ lhs: ◌  │
┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
│ Var     │╶┘ └─────────┘ └╴│ BinOp   │
│ name: a │                 │ op: Add │
└─────────┘               ┌╴│ lhs: ◌  │
              ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
              │ Var     │╶┘ └─────────┘ └╴│ BinOp   │
              │ name: b │                 │ op: Add │
              └─────────┘               ┌╴│ lhs: ◌  │
                            ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
                            │ BinOp   │╶┘ └─────────┘ └╴│ Var     │
                            │ op: Add │                 │ name: e │
                          ┌╴│ lhs: ◌  │                 └─────────┘
              ┌─────────┐ │ │ rhs: ◌  │╶┐ ┌─────────┐
              │ Var     │╶┘ └─────────┘ └╴│ Index   │
              │ name: c │               ┌╴│ arr: ◌  │
              └─────────┘   ┌─────────┐ │ │ ind: ◌  │╶┐ ┌─────────┐
                            │ Var     │╶┘ └─────────┘ └╴│ LitInt  │
                            │ name: d │                 │ val: 0  │
                            └─────────┘                 └─────────┘
```

It is important to understand the distinction between the AST and token trees. When writing macros,
you have to deal with *both* as distinct things.

One other aspect of this to note: it is *impossible* to have an unpaired paren, bracket or brace;
nor is it possible to have incorrectly nested groups in a token tree.

[tokenization]: https://en.wikipedia.org/wiki/Lexical_analysis#Tokenization
[reserved]: https://doc.rust-lang.org/reference/keywords.html#reserved-keywords
[`rustc_lexer`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_lexer
[`rustc_parse`]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_parse
[lexer]: https://github.com/rust-lang/rust/tree/master/compiler/rustc_parse/src/lexer
[Abstract Syntax Tree]: https://en.wikipedia.org/wiki/Abstract_syntax_tree
