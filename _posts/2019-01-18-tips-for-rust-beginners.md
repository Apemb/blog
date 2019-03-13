---
fn layout: post
author: thomas_wickham
title:  "Tips for Rust beginners"
date:   2019-01-18 22:00:00 +0100
categories: rust coding
toc: true
---

As you may know, Rust is one of my favorite tools. I've been writing code in Rust since 2015 and I can say that I got my fair part of quirks, misunderstandings, and errors.

I also enjoy teaching Rust to my curious friends and colleagues. In this post, I'd like to share with you the usual tips I dispense for them to start with less hassle.

_Thanks to [Nitnelave](https://github.com/nitnelave) for his multiple reviews._

## General Tips

### Use Rustup

The Rust toolchain installer is named rustup, and it works so well I don't see any reason not to use it. See [rustup.rs](https://rustup.rs/).

### Use Rust stable and avoid nightly

As you have rustup, you should use `rustup install stable` and work with Rust stable.

Even if Rust nightly has a lot of bells and whistles, don't give in to them. You are a beginner and the stable language is large enough. If you need something in nightly you probably are over-engineering your problem, want to [shave a yak](https://www.hanselman.com/blog/YakShavingDefinedIllGetThatDoneAsSoonAsIShaveThisYak.aspx), or misunderstood what you want to do. I encourage you to research your problem a step further and look for stable solutions.

### Use clippy

[Clippy](https://github.com/rust-lang/rust-clippy) is a very nice tool to learn best practices in canonic Rust. Put somewhere a shortcut to run `cargo +nightly clippy`easily and often.

It may not work, in which case I recommend to abandon this tip and don’t care too much. It's more of a “_nice to have_”.

### Use Cargo fmt frequently too see how rust code should look

[Rustfmt](https://github.com/rust-lang/rustfmt#quick-start) is a formatting tool for Rust source code. It rewrites your code in a way that is commonly agreed upon in the community, and avoids any coding-style war.

It's also nice to apprehend how Rust code should look like, so that you can mimic it in your code.

As soon as Rustfmt is installed, you can use `cargo fmt` to reformat your code.

### Use very few dependencies

You simply don't know what the quality of each dependency is. As you are a beginner, you don't have the tools to see what a quality library is. Popularity is a good metric, but it won't help you much as soon as you need to do something a bit niche.

Do it yourself, you will learn something on the way.

### Red Green Refactor: do TDD

Rust comes with [integrated unit testing](https://doc.rust-lang.org/book/ch11-01-writing-tests.html). You can and should abuse it !

I have talked about TDD in Rust in Zurich's RustFest 2017: [talk](https://www.youtube.com/watch?v=U3F7uAOCjEo), [slides](https://slides.com/thomaswickham/efficient-tdd-in-rust).

Example:

```rust
fn function(_arg: u32) -> u32 {
    12
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn it_should_return_3_when_called_with_12() {
        assert_eq!(3, function(12)) // run with `cargo test` to see it fail
    }
}
```

To learn more, let's look at what the [Rust book says about tests](https://doc.rust-lang.org/1.30.0/book/second-edition/ch11-03-test-organization.html).

## Design tips

### Your struct should always inherit Debug. Also partialEq and Eq

This one is simple. The`Debug` trait (kinda like interfaces) is useful for debug printf, [the `dbg!` macro](https://doc.rust-lang.org/std/macro.dbg.html), asserts, errors, and libraries. You should always, always derive it. Especially if you make a library, your users will need it to understand what happens.

[See the doc for more infos](https://doc.rust-lang.org/std/fmt/trait.Debug.html).

### Better be too open than too private

Too often I see a library with private types, or private constructors. Why not ? After all, that's why we have a privacy system !

Let's ask the question: what do you lose by opening too many types ? You could have leaky abstractions, broken contracts. Not great, but any non-mature library will have it anyway. With documentation, builders, and code contracts you can avoid it quite easily.

What do you get by having too many private types ? Untestable code. Broken documentation for your users. Undebuggable magic which may panic but that I can't see. The trade-off is far worse, so there is my easy choice.

In doubt: be public. It's no big deal to open your types.

### As a rule of thumb use String, not &str

*Note: Clippy will sometimes disagree. You can follow its advice, but don't be afraid to revert to `String` if you feel the need for it.*

This one is a massive tip. There is two types for string representation, which often confuse beginners. Often, they assume that `&str` is the simpler and so should be the default. _It's the reverse: `String` is simpler and should be the default choice._

What's the difference, would you say ? `String` is the owned and allocated version of `&str`. When the former resides in RAM, the latter can be anywhere: it can be a reference to immutable static data on a file, on a segment of read-only memory, or a pointer to a foreign callee. You don't know, you can't touch it, only see it. Thus, you can't do much with it, and the compiler will watch you closely.

Another important thing is _you won't lose any features by using a String_. It's even the opposite: allocating a `String` gives you more power as now you can change your string, pass it around.

Use `String` first. Then, when your code begin to stabilize, you will be able to spot easy replacements from `String` to `&str` without loss (it will be easy: does it compiles ?).

_NOTE: You can make a `String` from a `&str` with `String::from(other_string)`._

### &str for Input, String for output

You don't want to mutate the input arguments, right ? Then it's probably safe to take `&str` as input arguments.

But unless you know what you are doing, you shouldn't return a `&str`. Use `String` as return type so that

If you want to know why [this advanced explanation can help](https://stackoverflow.com/questions/23981391/how-exactly-does-the-callstack-work).

### Use clone() at will !

Don't be afraid to `.clone()` away your borrowing errors.

It's really not an issue as Rust code tends to be very sensible in term of memory consumption. If you measure that your memory consumption is high, you can find the hot spot and optimize it later.

### Never have &´a in your structs, always own your data

`&'a` in your struct means that the struct has a view in the data of another struct. You should read it as a `&`, then a `‘a`.

`&` means that it's an immutable reference and the `'a` is a name. The name of the lifetime of this other struct you want to look into. This name is useful to the compiler to prove that your struct instances will never outlive the struct they want to look into.

What's the matter ? Simple: if one of your struct holds a reference to data of another struct and it outlives the other structs (meaning that the other struct has been freed), it would mean that the other struct’s data is now garbage, something completely different. We do not want that. That's called a “_Use after free_” error and this often means at least a segfault of the program (immediate termination from the kernel because of a bad memory access). In the worst cases, your program _does not_ segfault and then it may be open to an exploitation by a malicious code.

The Rust compiler prevents these kind of errors by tracking lifetimes, and these `'a` are names of the lifetimes Rust tracks. The less you see them, the simpler your life is.

The best way not to see them is to own your data. For that, use owned types (`String` over `&str`), don't use references, and don't be afraid to clone the data when constructing your structs.

### Prefer templating to dynamic traits (boxed traits)

Here is some code with templates:

```rust
// using straight templates
fn to_string<TS: ToString>(arg: TS) -> String {
    arg.to_string()
}

// using impl traits
fn to_string2(arg: impl ToString) -> String {
    arg.to_string()
}
```

And here is a code with a dynamic trait:

```rust

fn to_string(arg: &dyn ToString) -> String {
    arg.to_string()
}
```

So what's the difference ? Not much for you. In both case your function has access to the same methods. In both cases you don't know what is the original type.

For the compiler it's another story. The template-way to write things will be unrolled for each use of the function (this action is named _monomophisation_), and may help rustc compute the lifetimes of your variables.

I tend to recommend to use templates as much as possible as they have few negative trade-offs and will make clearers and fewer errors.

_Note1: before Rust 2018, `&dyn ToString` was written `&ToString` and was named a boxed trait_

_Note2: `impl ToString` is an advanced part of the language that I don't recommend you to use until you need it, which is when you need to return a type that you don't know yet but implements a trait. Futures are a good example of this. This makes messy errors that are hard to debug if your are new to Rust._

### If rustc tells you “unknown type size”, try templating. If it don’t work, use reference

I won't go into details here but you can have this kind of errors:

```rust
fn my_func(array: [u8]) {
//         ^^^^^ doesn't have a size known at compile-time
}
```

You simply have to add a reference like that:

```rust
fn my_func(array: &[u8]) {}
```

Or make the type known at compile-time with a template:

```rust
fn my_func1<Array>(array: Array) {}
```

### Methods: first `&self`, then `&mut self`, then `self`

When you are making a method:

```rust
struct Foo;
impl Foo {
    fn method0() {}          // 0: no self, it's a static function
    fn method1(&self) {}     // 1: &self: method borrowing by reference
    fn method2(&mut self) {} // 2: &mut self: method borrowing mutably
    fn method3(self) {}      // 3: self: method consuming itself
}
```

`method0` is not a method, so don't forget to add the special `self` argument.

`method1` is borrowing the instance for a read-only use of the attributes. Nothing to see here.

`method2` is the same, but you can change (mutate) the attributes of the instance. Rustc will make sure that only one `&mut` to your instance is hold, which don't means much here so it's no big deal. The compiler will tell you when you need to add the `mut`. With time, you will be able to see them before the compiler errors.

`method3` consume itself, what it means is that you won't be able to use your object (or struct instance) after. This is useful only for unsafe operations or when you are a Rust wizard building a very fancy API. In which case you don't really need to read this article and your feedback is appreciated. :)
You should never have the use accidentally of `method3(self)`. Either it's by design and you know why you are doing it, either you didn't designed it as such and in this case you can safely avoid it.

### Never ever use Rc, Arc, Cell, RefCell, UnsafeCell. If you do, you are probably doing it wrong

You may need them, but most probably you are making an unsafe design and are currently fighting the borrow-checker.

If you are safe-gating a simple primitive type, you should use [the atomic package](https://doc.rust-lang.org/std/sync/atomic/) which will give you a thread-safe API over the data.

[expand the description why is it fighting the borrowck ? how to solve ?]

### Atomics (std::sync::atomic) are great. Use them at will

Atomics are special types that are supported by all the classic CPU and provide special garnatees around thread-safety.

If you ever need to design a concurrent system, I highly encourage you to look at the [the std::sync::atomic package](https://doc.rust-lang.org/std/sync/atomic/) and use it as a base primitive for your structs.

### Learn the multiple methods of Result and Option and use them

These two types are really great and provides lots and lots of useful methods for composability.

Here are handful links to the doc: [the std::result module](https://doc.rust-lang.org/std/result/index.html) gives nice information about the usage, but the real nuggets are in [the Result type documentation](https://doc.rust-lang.org/std/result/enum.Result.html). It's the same for [the std::option module](https://doc.rust-lang.org/std/option/index.html) and [the Option type documentation](https://doc.rust-lang.org/std/option/enum.Option.html).

Take your time to dive into it. Here is a nice example of Result composability: [(Source, the whole article is a gold mine which I encourage you to read)](https://blog.burntsushi.net/rust-error-handling/#composing-option-and-result)

```rust
use std::env;

// Result<OkType, ErrorType> is here either an int, either an error String
fn double_arg(mut argv: env::Args) -> Result<i32, String> {
    argv.nth(1)
        .ok_or("Please give at least one argument".to_owned())
        .and_then(|arg| arg.parse::<i32>().map_err(|err| err.to_string()))
}

fn main() -> Result<(), Error> {
    let n = double_arg(env::args())?;
    println!("{}", n)
}
```

### Prefer Vec\<T\> rather than raw arrays

They own their data, can grow, and as such are far easier to work with. And they have many many [handy methods](https://doc.rust-lang.org/std/vec/struct.Vec.html).

Do I really need to expand ?

### Don't return an Iterator, a Vec is easier

You may want to use the powerful iterators and profit from all their functions (lazyness, composition, performance, zero-cost).

The thing is, returning an iterator is an advanced topic. Try it for fun if you may, but it's not easy to deal to deal with the errors. Allocating a vector is probably fine so you should try that first.

Note that you can convert any Iterator to a Vector with:

```rust
let vector = some_iterator.collect()
```

And any vector to an iterator with the `.into_iter()`, `.iter()`, and `.iter_mut()` functions.

### The difference between `.into_iter()`, `.iter()`, and `.iter_mut()`

Do you remember the difference between `self`, `&self`, and `&mut self` in a method ? It's the same.

Rust has the convention to name `into_` something that will take `self`, do nothing for `&self` (as it's the default), and `_mut` for `&mut self` . It's the same for the iterators.

Note that the for loops consume iterators, so you should use the correct iterator over a vector.

```rust
// iterator `self` which consume the values
vector.into_iter()
      .map(|value| println!("{}", value))
      .collect();
// this is strictly the same as:
for value in vector {
    println!("{}", value);
}

// iterator `&self` which takes by reference
vector.iter()
      .map(|value| println!("{}", value))
      .collect()
// this is strictly the same as:
for value in &vector {
    println!("{}", value);
}

// iterator `&mut self` which takes by mutable reference
vector.iter_mut()
      .map(|value| println!("{}", value))
      .collect()
// this is strictly the same as:
for value in &mut vector {
    println!("{}", value);
}
```

### Learn the From trait and the conversion methods

Rust is strongly typed, and types are easy to create. That's why you want to be able to easily convert your data between types.

[Documentation of the From trait](https://doc.rust-lang.org/std/convert/trait.From.html)

There is some nice things with these traits like:

### Try and experiment with the Rust Playground

Rust has [a very nice interactive Playground](https://play.rust-lang.org/). You can use it to try things and tests dummy design that you think should work (or fail).

I often try to reduce my design errors with a sample code in the playground, then try to work around the issue.

# Wrapping it up

There are many other tips I could give, but that would be for another article.

Please remember to keep things simple and don’t add imaginary requirements. Indeed, performance is cool, but how do you know there is a problem if you can’t run your program ? How do you know that the code your are improving gives a better performance if you can’t measure it ?

In Rust, “_done is better than perfect_” is a mantra to follow. Especially with all the shiny features the language can provide.
