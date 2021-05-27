# 解析 Rust

在有些情况下解析某些 Rust [items] 会很有用。
这一章会展示一些能够解析 Rust 中更复杂的 items 的宏。

[items]:https://doc.rust-lang.org/nightly/reference/items.html

[transcribers]:https://doc.rust-lang.org/nightly/reference/macros-by-example.html

这些宏目的不是解析整个 items 语法，而是解析通用、有用的部分，
解析的方式也不会太复杂。
也就是说，我们不会涉及解析 *泛型* 之类的东西。

重点在于宏的匹配方式 (matchers) ；展开的部分 （ *Reference* 里使用的词语叫做 [transcribers] ），
仅仅用作例子，不需要特别关心它。

## 解析函数

```rust,editable
macro_rules! function_item_matcher {
    (

        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis fn $name:ident ( $( $arg_pat:ident : $arg_ty:ty ),* $(,)? )
    //                          ^~~~~~~~~~~~~~~~argument list~~~~~~~~~~~~~~~^
            $( -> $ret_ty:ty )?
    //      ^~~~return type~~~^
            { $($tt:tt)* }
    //      ^~~~~body~~~~^
    ) => {
        $( #[$meta] )*
        $vis fn $name ( $( $arg_name : $arg_ty ),* ) $( -> $ret_ty )? { $($tt)* }
    }
}

#function_item_matcher!(
#    #[inline]
#    #[cold]
#    pub fn foo(bar: i32, baz: i32, ) -> String {
#        format!("{} {}", bar, baz)
#    }
#);
#
# fn main() {
#     assert_eq!(foo(13, 37), "13 37");
# }
```

A simple function matcher that ignores qualifiers like `unsafe`, `async`, ... as well a generics and
where clauses. If parsing those is required it is likely that you are better off using a proc-macro
instead.

This lets you for example, inspect the function signature, generate some extra things from it and
then re-emit the entire function again. Kind of like a `Derive` proc-macro but weaker and for
functions.

> Ideally we would like to use a pattern fragment specifier instead of an ident for the arguments
> but this is currently not allowed. Fortunately people don't use patterns in function signatures
> that often so this is okay.

### Method

The macro for parsing basic functions is nice and all, but sometimes we would like to also parse
methods, functions that refer to their object via some form of `self` usage. This makes things a bit
trickier:

> WIP

## Struct

```rust
macro_rules! struct_item_matcher {
    // Unit-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident;
    ) => {
        $( #[$meta] )*
        $vis struct $name;
    };

    // Tuple-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident (
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~~^
                $field_vis:vis $field_ty:ty
    //          ^~~~~~a single field~~~~~~^
            ),*
        $(,)? );
    ) => {
        $( #[$meta] )*
        $vis struct $name (
            $(
                $( #[$field_meta] )*
                $field_vis $field_ty
            ),*
        );
    };

    // Named-Struct
    (
        $( #[$meta:meta] )*
    //  ^~~~attributes~~~~^
        $vis:vis struct $name:ident {
            $(
                $( #[$field_meta:meta] )*
    //          ^~~~field attributes~~~!^
                $field_vis:vis $field_name:ident : $field_ty:ty
    //          ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
            ),*
        $(,)? }
    ) => {
        $( #[$meta] )*
        $vis struct $name {
            $(
                $( #[$field_meta] )*
                $field_vis $field_name : $field_ty
            ),*
        }
    }
}

#struct_item_matcher!(
#    #[derive(Copy, Clone)]
#    pub(crate) struct Foo { 
#       pub bar: i32,
#       baz: &'static str,
#       qux: f32
#    }
#);
#struct_item_matcher!(
#    #[derive(Copy, Clone)]
#    pub(crate) struct Bar;
#);
#struct_item_matcher!(
#    #[derive(Clone)]
#    pub(crate) struct Baz (i32, pub f32, String);
#);
#fn main() {
#    let _: Foo = Foo { bar: 42, baz: "macros can be nice", qux: 3.14, };
#    let _: Bar = Bar;
#    let _: Baz = Baz(2, 0.1234, String::new());
#}
```

# Enum

Parsing enums is a bit more complex than structs so we will finally make use of some of the
[patterns] we have discussed, [Incremental TT Muncher] and [Internal Rules]. Instead of just
building the parsed enum again we will merely visit all the tokens of the enum, as rebuilding the
enum would require us to collect all the parsed tokens temporarily again via a
[Push Down Accumulator].

```rust
macro_rules! enum_item_matcher {
    // tuple variant
    (@variant $variant:ident (
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~~^
            $field_vis:vis $field_ty:ty
    //      ^~~~~~a single field~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    ) $(, $($tt:tt)* )? ) => {
        
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // named variant
    (@variant $variant:ident {
        $(
            $( #[$field_meta:meta] )*
    //      ^~~~field attributes~~~!^
            $field_vis:vis $field_name:ident : $field_ty:ty
    //      ^~~~~~~~~~~~~~~~~a single field~~~~~~~~~~~~~~~^
        ),* $(,)?
    //∨~~rest of input~~∨
    } $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // unit variant
    (@variant $variant:ident $(, $($tt:tt)* )? ) => {
        // process rest of the enum
        $( enum_item_matcher!(@variant $( $tt )*) )?
    };
    // trailing comma
    (@variant ,) => {};
    // base case
    (@variant) => {};
    // entry point
    (
        $( #[$meta:meta] )*
        $vis:vis enum $name:ident {
            $($tt:tt)*
        }
    ) => {
        enum_item_matcher!(@variant $($tt)*)
    };
}

#enum_item_matcher!(
#    #[derive(Copy, Clone)]
#    pub(crate) enum Foo { 
#        Bar,
#        Baz,
#    }
#);
#enum_item_matcher!(
#    #[derive(Copy, Clone)]
#    pub(crate) enum Bar {
#        Foo(i32, f32),
#        Bar,
#        Baz(),
#    }
#);
#enum_item_matcher!(
#    #[derive(Clone)]
#    pub(crate) enum Baz {}
#);
```

[patterns]:/patterns.html
[Push Down Accumulator]:/patterns/push-down-acc.html
[Internal Rules]:/patterns/internal-rules.html
[Incremental TT Muncher]:/patterns/tt-muncher.html
