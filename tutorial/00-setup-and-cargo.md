# 00: Setup, Cargo and the toolchain

> Official reading: [The Book, ch. 1 "Getting Started"](https://doc.rust-lang.org/book/ch01-00-getting-started.html)
> and [The Cargo Book, "Getting Started"](https://doc.rust-lang.org/cargo/getting-started/index.html)

## 1. The toolchain: three tools

| Tool | Role |
|------|------|
| `rustup` | Installs and updates Rust itself (`rustup update`) |
| `rustc` | The compiler. You rarely call it directly. |
| `cargo` | Build tool and package manager. **You use it for almost everything.** |

Check what you have:

```bash
rustc --version
cargo --version
rustup component add clippy rustfmt   # linter + formatter (usually already installed)
```

For your editor, install the **rust-analyzer** extension in VS Code. It gives
inline types, errors as you type, and go-to-definition. Turn on its "inlay hints".
Seeing the type the compiler inferred is a fast way to learn.

## 2. Your project, explained

```
service-practice/
├── Cargo.toml      ← the "manifest": name, version, edition, dependencies
├── Cargo.lock      ← exact versions of every dependency (created on first build)
├── src/
│   └── main.rs     ← the binary's entry point: fn main()
└── target/         ← build output (ignored by git)
```

`Cargo.toml`:

```toml
[package]
name = "service-practice"
version = "0.1.0"
edition = "2024"      # the language edition: a set of syntax rules, not a compiler version

[dependencies]        # crates (libraries) from https://crates.io go here
```

**Editions** (2015, 2018, 2021, 2024) let Rust evolve without breaking old code.
Crates from different editions work together. You're on the newest edition, 2024.

## 3. The Cargo commands you will use every day

```bash
cargo build            # compile (debug mode, fast to compile, slower to run)
cargo run              # build + run src/main.rs
cargo run --example x  # build + run examples/x.rs
cargo check            # type-check only, no binary: the FASTEST feedback loop
cargo test             # run all tests
cargo clippy           # lints: "here's a more idiomatic way"
cargo fmt              # auto-format all code
cargo build --release  # optimized build → target/release/service-practice
cargo add serde        # add a dependency to Cargo.toml
cargo doc --open       # build HTML docs for your code AND all its dependencies
```

## 4. Hello, world, and what each part means

Open `src/main.rs`:

```rust
fn main() {
    println!("Hello, world!");
}
```

- `fn main()`: the entry point of every binary.
- `println!`: the `!` means it's a **macro**, not a function. Macros generate code
  at compile time. `println!` checks your format string *at compile time*, so
  `println!("{}")` with a missing argument is a compile error, not a runtime crash.
- `;` ends a statement. (This matters more than in other languages. See chapter 01.)

Run it:

```bash
cargo run
```

## 5. Set up your examples folder

Create the folder where all lesson code will go:

```bash
mkdir examples
```

Then create `examples/00_hello.rs`:

```rust
// file: examples/00_hello.rs
fn main() {
    let name = "Rustacean";
    println!("Hello, {name}!");          // inline variable capture
    println!("Hello, {}!", name);        // positional argument
    println!("Debug view: {:?}", name);  // {:?} = Debug formatting (shows quotes)
    eprintln!("this goes to stderr");
}
```

```bash
cargo run --example 00_hello
```

## 6. How compilation works (the mental model)

Rust is **compiled ahead of time** into a native machine-code binary. There is no
VM and no garbage collector. Memory is freed at points the compiler decides, based
on rules you will learn in chapter 02.

So the compiler does a lot of checking up front: types, memory safety, thread
safety. Compiling is slower than in Go, and the compiler can feel strict. In return,
a program that compiles has **no null pointer exceptions, no use-after-free, and no
data races**, by construction. You'll often hear Rustaceans say "if it compiles,
it usually works".

## ✅ Exercises

1. Run `cargo build --release` and compare the size and location of the debug
   binary (`target/debug/`) and the release binary (`target/release/`).
2. Add `println!("{}", 1 + );` to `00_hello.rs`, run `cargo check`, and read the
   error. Then remove it.
3. Run `cargo doc --open` and look around. Every crate on crates.io has docs like
   these on https://docs.rs.

Next → [01: Basics](01-basics.md)
