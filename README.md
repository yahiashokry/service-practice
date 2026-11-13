# 🦀 Rust: from zero to your first running service

A hands-on Rust tutorial that follows the **official Rust documentation** and
ends with a working **REST API service** built with [Tokio](https://tokio.rs) and
[axum](https://docs.rs/axum).

It has two goals:

1. **Understand how Rust works**: ownership, borrowing, traits, lifetimes,
   concurrency and async, explained by *why* Rust is designed that way, not just
   the syntax.
2. **Build your first service**: a Todo API with JSON, shared state, error
   handling, logging, tests and graceful shutdown.

## 📚 Chapters

| # | Chapter | Official source |
|---|---------|-----------------|
| 00 | [Setup, Cargo and the toolchain](tutorial/00-setup-and-cargo.md) | The Book ch. 1, The Cargo Book |
| 01 | [Basics: variables, types, functions, control flow](tutorial/01-basics.md) | The Book ch. 3 |
| 02 | [Ownership, borrowing and slices](tutorial/02-ownership-borrowing.md) ⭐ | The Book ch. 4 |
| 03 | [Structs, enums and pattern matching](tutorial/03-structs-enums-matching.md) | The Book ch. 5, 6, 19 |
| 04 | [Modules, crates and tests](tutorial/04-modules-crates-tests.md) | The Book ch. 7, 11 |
| 05 | [Collections and error handling](tutorial/05-collections-errors.md) | The Book ch. 8, 9 |
| 06 | [Generics, traits and lifetimes](tutorial/06-generics-traits-lifetimes.md) | The Book ch. 10 |
| 07 | [Closures and iterators](tutorial/07-closures-iterators.md) | The Book ch. 13 |
| 08 | [Smart pointers and concurrency](tutorial/08-smart-pointers-concurrency.md) | The Book ch. 15, 16 |
| 09 | [Async Rust and Tokio](tutorial/09-async-tokio.md) | The Book ch. 17, Tokio tutorial |
| 10 | [Build the service](tutorial/10-build-the-service.md) 🚀 | axum, serde, tracing docs |

Each chapter explains the concepts and gives you a program to **type yourself**.
It also shows compiler errors to trigger on purpose so you learn to read them,
and ends with **exercises** (solutions are in collapsible blocks). The
[tutorial README](tutorial/README.md) has the full roadmap and the list of
official resources.

## 🚀 Getting started

**Prerequisites:** Rust installed through [rustup](https://rustup.rs). The code
uses the 2024 edition, so you need Rust 1.85 or newer.

```bash
git clone https://github.com/yahiashokry/service-practice.git
cd service-practice
cargo run          # prints "Hello, world!"
```

Then open [`tutorial/00-setup-and-cargo.md`](tutorial/00-setup-and-cargo.md) and
work through the chapters in order.

Practice code goes in `examples/`, one file per lesson. Run a lesson with:

```bash
cargo run --example 01_basics
```

`src/` is kept for the service you build in chapter 10.

## 🛠️ The service you'll build

A Todo REST API with in-memory storage:

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/health` | Liveness check |
| `GET` | `/todos` | List all todos |
| `POST` | `/todos` | Create a todo: `{"title": "..."}` |
| `GET` | `/todos/{id}` | Get one todo |
| `PATCH` | `/todos/{id}` | Update: `{"title"?: "...", "done"?: true}` |
| `DELETE` | `/todos/{id}` | Delete a todo |

Once you've finished chapter 10:

```bash
cargo run
curl -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"title":"learn ownership"}'
curl localhost:3000/todos
```

**Stack:** `tokio` (async runtime), `axum` (web framework), `serde` (JSON),
`thiserror` (error types), `tracing` and `tower-http` (logging).

## 📖 Official resources

- [The Rust Programming Language](https://doc.rust-lang.org/book/) ("The Book")
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://rustlings.rust-lang.org/)
- [The Standard Library docs](https://doc.rust-lang.org/std/)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial)
- [axum documentation](https://docs.rs/axum/latest/axum/)

## 🤝 Contributing

Found a typo, an outdated example, or an explanation that could be clearer?
Issues and pull requests are welcome.
