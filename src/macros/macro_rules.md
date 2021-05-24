# `macro_rules!`

有了这些知识，我们终于可以引入 `macro_rules!` 了。
如前所述，`macro_rules!` 本身就是一个语法扩展，
也就是说它并不是 Rust 语法的一部分。它的形式如下：

```rust,ignore
macro_rules! $name {
    $rule0 ;
    $rule1 ;
    // …
    $ruleN ;
}
```

*至少得有一条规则* (rule) ，最后一条规则后面的分号可被省略。
规则里你可以使用大/中/小括号：`{}`、`[]`、`()`。
调用宏时使用 `{ .. }` 或者 `( ... );` 形式（带分号或者不带分号）。
注意最后的分号 *总是* 会被解析成一个条目 (item)。

每条“规则”都形如：

```ignore
    ($matcher) => {$expansion}
```

如前所述，分组符号可以是任意一种括号，在模式匹配外侧使用小括号、表达式外侧使用大括号只是出于传统。

如果你好奇的话，`macro_rules!` 的调用将被展开为   *空* 。
至少，在 AST 中它被展开为空。
它所影响的是编译器内部的结构，以将该宏注册 (register) 进去。
因此，技术上讲你可以在任何一个空展开合法的位置使用 `macro_rules!` 。

## 模式匹配

当一个宏被调用时，对应的 `macro_rules!` 解释器将按照声明顺序一一检查规则。
对每条规则，它都将尝试将输入标记树的内容与该规则的进行匹配。
某个模式必须与输入 *完全* 匹配才被认为是一次匹配。（这里所译“模式”的原词叫 matcher）

如果输入与某个模式相匹配，则该调用项将被相应的展开内容所取代；
否则，将尝试匹配下条规则。
如果所有规则均匹配失败，则宏展开会失败并报错。

最简单的例子是空模式：

```rust,ignore
macro_rules! four {
    () => { 1 + 3 };
}
```

它将且仅将匹配到空的输入，即 `four!()`、`four![]` 或 `four!{}` 。

注意调用所用的分组标记并不需要匹配定义时采用的分组标记。也就是说，你可以通过four![]调用上述宏，此调用仍将被视作匹配。只有调用时的输入内容才会被纳入匹配考量范围。

模式中也可以包含字面标记树。这些标记树必须被完全匹配。将整个对应标记树在相应位置写下即可。比如，为匹配标记序列4 fn ['spang "whammo"] @_@，我们可以使用：

This matches if and only if the input is also empty (*i.e.* `four!()`, `four![]` or `four!{}`).

Note that the specific grouping tokens you use when you invoke the macro *are not* matched. That is,
you can invoke the above macro as `four![]` and it will still match. Only the *contents* of the
input token tree are considered.

Matchers can also contain literal token trees, which must be matched exactly. This is done by simply
writing the token trees normally. For example, to match the sequence `4 fn ['spang "whammo"] @_@`,
you would write:

```rust,ignore
macro_rules! gibberish {
    (4 fn ['spang "whammo"] @_@) => {...};
}
```

You can use any token tree that you can write.

## Metavariables

Matchers can also contain captures. These allow input to be matched based on some general grammar
category, with the result captured to a metavariable which can then be substituted into the output.

Captures are written as a dollar (`$`) followed by an identifier, a colon (`:`), and finally the
kind of capture which is also called the fragment-specifier, which must be one of the following:

* `block`: a block (i.e. a block of statements and/or an expression, surrounded by braces)
* `expr`: an expression
* `ident`: an identifier (this includes keywords)
* `item`: an item, like a function, struct, module, impl, etc.
* `lifetime`: a lifetime (e.g. `'foo`, `'static`, ...)
* `literal`: a literal (e.g. `"Hello World!"`, `3.14`, `'🦀'`, ...)
* `meta`: a meta item; the things that go inside the `#[...]` and `#![...]` attributes
* `pat`: a pattern
* `path`: a path (e.g. `foo`, `::std::mem::replace`, `transmute::<_, int>`, …)
* `stmt`: a statement
* `tt`: a single token tree
* `ty`: a type
* `vis`: a possible empty visibility qualifier (e.g. `pub`, `pub(in crate)`, ...)

For more in-depth description of the fragement specifiers, check out the
[Fragment Specifiers](./minutiae/fragment-specifiers.md) chapter.

For example, here is a `macro_rules!` macro which captures its input as an expression under the
metavariable `$e`:

```rust,ignore
macro_rules! one_expression {
    ($e:expr) => {...};
}
```

These metavariables leverage the Rust compiler's parser, ensuring that they are always "correct". An
`expr` metavariables will *always* capture a complete, valid expression for the version of Rust being
compiled.

You can mix literal token trees and metavariables, within limits ([explained below]).

To refer to a metavariable you simply write `$name`, as the type of the variable is already
specified in the matcher. For example:

```rust,ignore
macro_rules! times_five {
    ($e:expr) => { 5 * $e };
}
```

Much like macro expansion, metavariables are substituted as complete AST nodes. This means that no
matter what sequence of tokens is captured by `$e`, it will be interpreted as a single, complete
expression.

You can also have multiple metavariables in a single matcher:

```rust,ignore
macro_rules! multiply_add {
    ($a:expr, $b:expr, $c:expr) => { $a * ($b + $c) };
}
```

And use them as often as you like in the expansion:

```rust,ignore
macro_rules! discard {
    ($e:expr) => {};
}
macro_rules! repeat {
    ($e:expr) => { $e; $e; $e; };
}
```

There is also a special metavariable called [`$crate`] which can be used to refer to the current
crate.

[`$crate`]:./minutiae/hygiene.html#crate

## Repetitions

Matchers can contain repetitions. These allow a sequence of tokens to be matched. These have the
general form `$ ( ... ) sep rep`.

* `$` is a literal dollar token.
* `( ... )` is the paren-grouped matcher being repeated.
* `sep` is an *optional* separator token. It may not be a delimiter or one
    of the repetition operators. Common examples are `,` and `;`.
* `rep` is the *required* repeat operator. Currently, this can be:
    * `?`: indicating at most one repetition
    * `*`: indicating zero or more repetitions
    * `+`: indicating one or more repetitions

    Since `?` represents at most one occurrence, it cannot be used with a separator.

Repetitions can contain any other valid matcher, including literal token trees, metavariables, and
other repetitions allowing arbitrary nesting.

Repetitions use the same syntax in the expansion and repeated metavariables can only be accessed
inside of repetitions in the expansion.

For example, below is a macro which formats each element as a string. It matches zero or more
comma-separated expressions and expands to an expression that constructs a vector.

```rust
macro_rules! vec_strs {
    (
        // Start a repetition:
        $(
            // Each repeat must contain an expression...
            $element:expr
        )
        // ...separated by commas...
        ,
        // ...zero or more times.
        *
    ) => {
        // Enclose the expansion in a block so that we can use
        // multiple statements.
        {
            let mut v = Vec::new();

            // Start a repetition:
            $(
                // Each repeat will contain the following statement, with
                // $element replaced with the corresponding expression.
                v.push(format!("{}", $element));
            )*

            v
        }
    };
}

fn main() {
    let s = vec_strs![1, "a", true, 3.14159f32];
    assert_eq!(s, &["1", "a", "true", "3.14159"]);
}
```

You can repeat multiple metavariables in a single repetition as long as all metavariables repeat
equally often. So this invocation of the following macro works:

```rust
macro_rules! repeat_two {
    ($($i:ident)*, $($i2:ident)*) => {
        $( let $i: (); let $i2: (); )*
    }
}

repeat_two!( a b c d e f, u v w x y z );
```

But this does not:

```rust
# macro_rules! repeat_two {
#     ($($i:ident)*, $($i2:ident)*) => {
#         $( let $i: (); let $i2: (); )*
#     }
# }

repeat_two!( a b c d e f, x y z );
```

failing with the following error

```
error: meta-variable `i` repeats 6 times, but `i2` repeats 3 times
 --> src/main.rs:6:10
  |
6 |         $( let $i: (); let $i2: (); )*
  |          ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

&nbsp;

For the complete grammar definition you may want to consult the 
[Macros By Example](https://doc.rust-lang.org/reference/macros-by-example.html#macros-by-example)
chapter of the Rust reference.
