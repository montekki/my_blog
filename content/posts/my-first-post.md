---
title: "Procedural macros in Rust"
date: 2019-02-24T14:12:31+03:00
draft: true
---

# Metaprogramming in Rust

Howdy ya'll. Today we will be looking into the
[procedural](https://blog.rust-lang.org/2018/12/21/Procedural-Macros-in-Rust-2018.html)
[macros](https://doc.rust-lang.org/1.30.0/book/2018-edition/appendix-04-macros.html) 
toolset of the [Rust](https://www.rust-lang.org/) programming language.

Rust language provides us with two types of metaprogramming tools:

* declarative macros
* procedural macros

Features ``println!`` or ``vec!`` are implemented using declarative macros and
features like ``derive`` traits are actually procedural macros.

Declarative macros (``macro_rules!``) operate in the declarative
pattern-matching manner, think of it as of feeding some input into a
``match`` expression and getting some output.

On the other hand, procedural macros allow one to implement a more agile
metaprogramming patterns by operating on Rust code and producing some
Rust code as output.

Let's try to dig deeper into the latter.

## Procedural macros

Rust's provides with a basic example of using procedural macros for
implementing a custome ``derive`` trait. Let's take a look at it with
slight modifications.

First off, we have a user of our marcos, that lives in the ``user`` repo
and has the following ``src/main.rs``:

{{< highlight rust >}}
extern crate my_macro;
extern crate my_macro_derive;

use my_macro::HelloMacro;
use my_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
{{< /highlight >}}

Here we see our type ``Pancakes`` and that it's equipped with a
``#[derive(HelloMacro)]`` macro to get a default implementation of
the ``hello_macro()`` function.

The trait we want to get the default implementation of lives in the
``my_macro`` crate and it defines the trait we want to implement:

Filename: src/lib.rs:

{{< highlight rust >}}
pub trait HelloMacro {
    fn hello_macro();
}
{{< /highlight >}}

To get the default implementation of this trait the procedural macro
is defined. It lives in it's own crate inside the ``my_macro``
crate. By convention for a crate named ``foo`` the derive macro
implementaion has to be called ``foo_derive``, so our derive macro
crate is called ``my_macro_derive``.

The ``my_macro_derive`` crate has to be declared as a procedural macro
crate in ``Cargo.toml``:

{{< highlight toml >}}
[lib]
proc-macro = true

[dependencies]
syn = "0.15"
quote = "0.6"
proc-macro2 = "0.4"
{{< /highlight >}}

To implement the procedural macro we will be using the following crates:

* ``proc_macro`` allows the user to convert Rust code into a string
containing that Rust code
* ``syn`` parses Rust code into some structures we can operate on
* ``quote`` takes the ``syn`` data structures with our modifications and
turns them back into Rust code.

Ok, now the implementation in the Rust Book goes almost exactly the following
way:

{{< highlight rust >}}
extern crate proc_macro;
extern crate quote;
extern crate syn;

use proc_macro::TokenStream;
use quote::quote;
use syn::DeriveInput;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    let ast = syn::parse(input).unwrap();

    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}", stringify!(#name));
            }
        }
    };
    gen.into()
}
{{< /highlight >}}

When we run the user we will see that our type ``Pancakes`` now
has the default implementation of the ``HelloMacro`` trait:

{{< highlight bash >}}
$ cargo run
Hello, Macro! My name is Pancakes
{{< /highlight >}}

Now, let's talk about this code. First of all, all the
necessary external creates are brought into scope.

Then, we see the function ``hello_macro_derive`` that is annotated
with a ``proc_macro_derive`` with the name ``HelloMacro`` specified,
this matches our trait name.

Important thing to understand is that this function will
get called during compile time, at the point where the user
has specified ``#[derive(HellMacro)]`` in his code.

## A bit more verbose

We see, that our ``hello_macro_derive`` function receives an input of
type ``proc_macro::TokenStream`` and returns a value of the same type.
If we check the [documentation](https://doc.rust-lang.org/proc_macro/struct.TokenStream.html) for this type, we find that values of this type may be cast to
``String``. Let's print the input and the output value of our function:

{{< highlight rust >}}
#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    dbg!(input.to_string());

    let ast = syn::parse(input).unwrap();

    let res = impl_hello_macro(&ast);
    dbg!(res.to_string());

    res
}
{{< /highlight >}}

and try to build our ``user`` crate:

{{< highlight bash >}}

[./my_macro/my_macro_derive/src/lib.rs:10] input.to_string() = "struct Pancakes;"
[./my_macro/my_macro_derive/src/lib.rs:14] res.to_string() = "impl HelloMacro for Pancakes {\nfn hello_macro (  ) {\nprintln ! ( \"Hello, Macro! My name is {}\" , stringify ! ( Pancakes ) ) ; } }"
    Finished dev [unoptimized + debuginfo] target(s) in 1.03s
     Running `target/debug/user`
Hello, Macro! My name is Pancakes
{{< /highlight >}}

Ok, it looks like the ``TokenStream`` we are receiving as input is the
``struct Pancakes;``, the type declaration that we've applied our ``derive``
macro. As an output we produce the ``TokenStream`` that represents a Rust
piece of code that implements the ``HelloMacro`` trait for the type ``Pancakes``
we have received as input.

What is done between these points in type is the following

* [``syn::parse``](https://docs.rs/syn/0.15.26/syn/fn.parse.html) is used
to parse the ``TokenStream`` value into ``DeriveInput`` struct and pass it
to ``impl_hello_marco``. In fact, ``syn::parse`` is a template function
and it can parse ``TokenStream`` into any type that implements
``syn::parse::Parse`` trait.
* [``syn::DeriveInput``](https://docs.rs/syn/0.15.26/syn/struct.DeriveInput.html)
is used to get the name of the identificator that is stored into variable ``name``
* ``name`` variable is substituted in the contents of the ``quote!`` macro
to get the final implementation of the ``HelloMacro`` trait.

``DeriveInput`` has the following declaration:

{{< highlight rust >}}
pub struct DeriveInput {
    pub attrs: Vec<Attribute>,
    pub vis: Visibility,
    pub ident: Ident,
    pub generics: Generics,
    pub data: Data,
}
{{< /highlight >}}

We see that ``ast.ident`` has type ``syn::Ident``. Now, if we take a look
at the [documentation](https://docs.rs/quote/0.6.11/quote/) for the
``quote::quote`` macro, we read the following about the interpolation of
types inside the ``quote!`` macro:

> The #var syntax performs interpolation of runtime variables into the quoted tokens.
> ``ToTokens`` Types that can be interpolated inside a quote! invocation.

``syn::Ident`` implements the ``quote::ToTokens`` trait so it can be
interpolated in the body of our ``quote`` macro.
