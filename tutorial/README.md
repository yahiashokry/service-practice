# Rust: from zero to your first running service

This tutorial is a guided path through the **official Rust documentation**. You do
the work inside this Cargo project (`service-practice`). Each chapter:

1. explains a topic and **why Rust works that way** (the mental model),
2. gives you code to **type yourself** (don't copy-paste; typing it is how you learn it),
3. shows the compiler errors you will hit, and how to read them,
4. ends with **exercises** (solutions are hidden in collapsible blocks),
5. links to the official chapter for deeper reading.

By the end you will have written a working REST API service (a todo API) with
`tokio` + `axum`, including JSON, shared state, error handling, logging, tests, and
graceful shutdown.

## The roadmap

| # | Chapter | Official source |
|---|---------|-----------------|
| 00 | [Setup, Cargo and the toolchain](00-setup-and-cargo.md) | The Book ch. 1, The Cargo Book |
| 01 | [Basics: variables, types, functions, control flow](01-basics.md) | The Book ch. 3 |
| 02 | [Ownership, borrowing and slices](02-ownership-borrowing.md) ⭐ the core of Rust | The Book ch. 4 |
| 03 | [Structs, enums and pattern matching](03-structs-enums-matching.md) | The Book ch. 5, 6, 19 |
| 04 | [Modules, crates and tests](04-modules-crates-tests.md) | The Book ch. 7, 11 |
| 05 | [Collections and error handling](05-collections-errors.md) | The Book ch. 8, 9 |
| 06 | [Generics, traits and lifetimes](06-generics-traits-lifetimes.md) | The Book ch. 10 |
| 07 | [Closures and iterators](07-closures-iterators.md) | The Book ch. 13 |
| 08 | [Smart pointers and concurrency](08-smart-pointers-concurrency.md) | The Book ch. 15, 16 |
| 09 | [Async Rust and Tokio](09-async-tokio.md) | The Book ch. 17, Tokio tutorial |
| 10 | [Build the service](10-build-the-service.md) 🚀 | axum docs, serde, tracing |

Plan on roughly one chapter per day or two. Chapter 02 deserves extra time. If
ownership "clicks", the rest of Rust follows from it.

## How to work through it

All the practice code goes into `examples/` in this project, one file per lesson.
Cargo treats every `examples/*.rs` file as a small separate program:

```bash
cd ~/projects/rust-projects/code/service-practice
cargo run --example 01_basics      # runs examples/01_basics.rs
```

`src/main.rs` stays for the real service, which you build in chapter 10.

Habits that pay off:

- **Read compiler errors fully.** Rust's errors are teaching tools. They often
  contain the exact fix. For a longer explanation of any error code, run
  `rustc --explain E0382`.
- **Break things on purpose.** When a chapter says "this does not compile", try it
  and read the error.
- Run `cargo clippy` often. It is a linter that teaches idiomatic Rust.
- Run `cargo fmt` to format your code automatically.

## The official resources this tutorial is built on

Keep these open in browser tabs. They are the best Rust material available, and
they are free.

| Resource | What it is | Link |
|----------|------------|------|
| **The Rust Programming Language** ("The Book") | The main official guide. This tutorial follows its order. | https://doc.rust-lang.org/book/ |
| **Rust by Example** | The same topics taught through small runnable examples | https://doc.rust-lang.org/rust-by-example/ |
| **Rustlings** | Small exercises that you fix until they compile. Pairs well with this tutorial. | https://rustlings.rust-lang.org/ |
| **The Standard Library docs** | Reference for `String`, `Vec`, `Option`, and the rest of `std` | https://doc.rust-lang.org/std/ |
| **The Cargo Book** | Everything about Cargo and `Cargo.toml` | https://doc.rust-lang.org/cargo/ |
| **The Rust Reference** | Precise language specification (for when you want the exact rules) | https://doc.rust-lang.org/reference/ |
| **Asynchronous Programming in Rust** | The official async book | https://rust-lang.github.io/async-book/ |
| **Tokio tutorial** | The standard async runtime, used by axum | https://tokio.rs/tokio/tutorial |
| **axum docs** | The web framework used in chapter 10 | https://docs.rs/axum/latest/axum/ |
| **Serde** | Serialization (JSON) framework | https://serde.rs/ |

Tip: `rustup doc --book` opens a local, offline copy of The Book, and
`rustup doc --std` does the same for the standard library docs.
