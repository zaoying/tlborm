# 卫生性

## 部分卫生的

Rust 里的 `macro_rules!` 是 **部分** 卫生的。
具体来说，对于绝大多数标识符，它是卫生的；
但对泛型参数和生命周期来说，它不是卫生的。

之所以能做到“卫生”，在于每个标识符都被赋予了一个看不见的“句法上下文”。
在比较两个标识符时，只有在标识符的原文名称和句法上下文都 **完全一样** 的情况下，
两个标识符才能被视作等同。

为阐释这一点，考虑下述代码：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

我们将采用背景色来表示句法上下文。现在，将上述宏调用展开如下：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> <span class="ident">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

首先，回想一下，在展开的期间调用 `macro_rules!` 宏，实际不会真正出现展开的结果。

其次，如果我们现在就尝试编译上述代码，编译器将报如下错误：

```text
error[E0425]: cannot find value `a` in this scope
  --> src/main.rs:13:21
   |
13 | let four = using_a!(a / 10);
   |                     ^ not found in this scope
```

注意到宏在展开后背景色（即其句法上下文）发生了改变。
每处宏展开均赋予其内容一个新的、独一无二的上下文。
故而，在展开后的代码中实际上存在 *两个* 不同的 `a`，它们分别有不同的句法上下文。
即，第一个 `a` 与第二个 `a` 并不相同，即使它们便看起来很像。

尽管如此，被替换进宏展开中的标记仍然 **保持** 着它们原有的句法上下文。
因为它们是被传给这宏的，并非这宏本身的一部分。
因此，我们作出如下修改：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="macro">macro_rules</span><span class="macro">!</span> <span class="ident">using_a</span> {&#xa;    (<span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span>:<span class="ident">ident</span>, <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>:<span class="ident">expr</span>) <span class="op">=&gt;</span> {&#xa;        {&#xa;            <span class="kw">let</span> <span class="macro-nonterminal">$</span><span class="macro-nonterminal">a</span> <span class="op">=</span> <span class="number">42</span>;&#xa;            <span class="macro-nonterminal">$</span><span class="macro-nonterminal">e</span>&#xa;        }&#xa;    }&#xa;}&#xa;&#xa;<span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> <span class="macro">using_a</span><span class="macro">!</span>(<span class="ident">a</span>, <span class="ident">a</span> <span class="op">/</span> <span class="number">10</span>);</span></pre>

展开如下：

<pre class="rust rust-example-rendered"><span class="synctx-0"><span class="kw">let</span> <span class="ident">four</span> <span class="op">=</span> </span><span class="synctx-1">{&#xa;    <span class="kw">let</span> </span><span class="synctx-0"><span class="ident">a</span></span><span class="synctx-1"> <span class="op">=</span> <span class="number">42</span>;&#xa;    </span><span class="synctx-0"><span class="ident">a</span> <span class="op">/</span> <span class="number">10</span></span><span class="synctx-1">&#xa;}</span><span class="synctx-0">;</span></pre>

因为只用了一种 `a`，编译器将欣然接受此段代码。

## `$crate` 元变量


宏需要原定义 (defining) crate 的其他 items 时，由于“卫生性”，我们需要使用 `$crate` 元变量。

这个特殊的元变量所做的事情是，它展开成宏所定义的 (defining) crate 的绝对路径。

```rust,ignore
//// 在 `helper_macro` crate 里定义 `helped!` 和 `helper!` 宏
#[macro_export]
macro_rules! helped {
    // () => { helper!() } // This might lead to an error due to 'helper' not being in scope.
    () => { $crate::helper!() }
}

#[macro_export]
macro_rules! helper {
    () => { () }
}

//// 在另外的 crate 中使用这两个宏
// 注意：`helper_macro::helper` 并没有导入进来
use helper_macro::helped;

fn unit() {
   // 这个宏能运行通过，因为 `$crate` 正确地展开成 `helper_macro` crate 的路径（而不是使用者的路径）
   helped!();
}
```

请注意，`$crate` 用在指明非宏的 items 时，它必须和完整且有效的模块路径一起使用。如下：

```rust
pub mod inner {
    #[macro_export]
    macro_rules! call_foo {
        () => { $crate::inner::foo() };
    }

    pub fn foo() {}
}
```
