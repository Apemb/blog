---
layout: post
title:  "Tips for Rust beginners"
date:   2019-01-18 22:00:00 +0100
categories: rust
---

As you may know, Rust is a pet peve of mine. I code in Rust since 2015 and I can say that I got my fair part of quirks, misunderstandings, and errors.

I also enjoy teaching Rust to my curious friends and colleagues. In this post, I'd like to share with you the usual tips I dispense for them to start with less hassle.


[TOC]


## General Tips

### Code in Rust stable

Rust nightly has a lot of bells and whistles. Don't. succomb to them. You are a beginner and the stable language is large enough. If you need something in nightly, I will probably challenge your need how your formulate it, and question the choice of Rust of the use-case.

### Use clippy

[Clippy](https://github.com/rust-lang/rust-clippy) is a very nice tool to learn best practices in canonic Rust. Put somewhere a shortcut to run `cargo +nightly clippy` oftentimes.

It may not work, in this case: abandon this tip and don’t care too much. It's more of a nice to have.

### Use Cargo fmt frequently too see how rust code should look

[Rustfmt](https://github.com/rust-lang/rustfmt#quick-start) is a formatting tool for Rust source code. It rewrites your code in a way that is commonly agreed on on the community, and kill any coding-style war instantly.

It's also nice to apprehend how Rust code should look like, so that you can mimic it in your code.

### Use very few dependencies

You simply don't know what the quality of this dependency is. As you are a beginner, you don't have the tools to see what a quality library is. Popularity is a good metric, but it won't help you much as soon as you need to do something a bit niche.

Do it yourself, you will learn something on the journey.

### Red Green Refactor: do TDD

Rust comes with [integrated unit testing](https://doc.rust-lang.org/book/ch11-01-writing-tests.html). You can and should abuse it ! 

I have talked about TDD in Rust in Zurich's RustFest 2017: [talk](https://www.youtube.com/watch?v=U3F7uAOCjEo), [slides](https://slides.com/thomaswickham/efficient-tdd-in-rust).

## Design tips

### Your struct should always inherit Debug. Also partialEq and Eq

This one is simple. `Debug` is usefull for debug printf, [the `dbg!` macro](https://doc.rust-lang.org/std/macro.dbg.html), and asserts. You shall always derive it. I don't see any valid reason not to. [See the doc for more infos](https://doc.rust-lang.org/std/fmt/trait.Debug.html).

### As a rule of thumb use String, not &str

This one is a massive tip. There is two types for string representation, which often confuse beginners. Often, they assume that `&str` is the simpler and so should be the default. _It's the reverse: `String` is simpler and should be the default choice._

What's the difference, would you say ? `String` is the owned and allocated version of `&str`. When the former resides in RAM, the latter can be anywhere: it can be a reference to immutable static data on a file, on a segment of read-only memory, or a pointer to a foreign callee. You don't know, you can't touch it, only see it. Thus, you can't do much with it, and the compiler will watch you thouroughly.

Another important thing is _you won't lose any features by using a String_. It's even the opposite: allocating a `String` gives you more power as now you can change your string, pass it around.

Use `String` first. Then, when your code begin to stabilize, you will be able to spot easy replacements from `String` to `&str` without loss (it will be easy: does it compiles ?).

_NOTE: You can make a `String` from a `&str` with `String::from(other_string)`._

### &str for Input, String for output

You don't want to mutate the input arguments, right ? Then it's probably safe to take `&str` as input arguments.

But unless you know what you are doing, you shouldn't return a `&str`. Use `String` as return type so that 

If you want to know why [this advanced explanation can help](https://stackoverflow.com/questions/23981391/how-exactly-does-the-callstack-work). 

### Use clone() at will !

Don't be affraid to `.clone()` away your borrowing errors. 

It's really not an issue as Rust code tends to be very sensible in term of memory consumption. If you measure that your memory consumption is high, you can find the hot spot and optimize it later.

### Never have &´a in your structs, always own your data

`&'a` in your struct means that the struct has a view in the data of another struct. You should read it as a `&`, then a `‘a`.

`&` means that it's an immutable reference and the `'a` is a name. The name of the lifetime of this other struct you want to look into. This name is useful to the compiler to prove that your struct instances will never outlive the struct they want to look into.

What's the matter ? Simple: if one of your struct holds a reference to data of another struct and it outlives the other structs (meaning that the other struct has been freed), it would mean that the other struct’s data is now garbage, something completely different. We do not want that. That's called a “Use after free” error and this often means at least a segfault of the program (immediate termination from the kernel because of a bad memory access). In the worst cases, your program _do not_ segfault and then it may be open to an exploitation by a malicious code.

The Rust compiler prevent these kind of errors by tracking lifetimes, and these `'a` are names of the lifetimes Rust tracks. The lesser you see them, the simpler your life is.

The best way to not see them is to own your data. For that, use owned types (`String` over `&str`), don't use references, and don't be afraid to clone the data on the construction of your struct.

# Wrapping it up

There are many other tips I could give, but that would be for another article.

Please remember to keep things simple and don’t add imaginary requirements. Indeed, performance is cool, but how do you know there is a problem if you can’t run your program ? How do you know that the code your are improving gives a better performance if you can’t measure it ?

In Rust, done is better than perfect is a mantra to follow. Especially with all the shiny features the langage can provide.