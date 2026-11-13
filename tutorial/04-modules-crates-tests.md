# 04: Modules, crates and tests

> Official reading: [The Book, ch. 7 "Packages, Crates, and Modules"](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html),
> [ch. 11 "Writing Automated Tests"](https://doc.rust-lang.org/book/ch11-00-testing.html),
> [ch. 14 "More about Cargo and Crates.io"](https://doc.rust-lang.org/book/ch14-00-more-about-cargo.html)

## 1. The vocabulary

| Term | Meaning |
|------|---------|
| **Package** | A Cargo project: one `Cargo.toml`. Yours is `service-practice`. |
| **Crate** | One compilation unit. Either a **binary** (has `main`) or a **library**. |
| **Crate root** | The file the compiler starts from: `src/main.rs` (binary) or `src/lib.rs` (library) |
| **Module** | A namespace inside a crate, used to organize code and control privacy |
| **Path** | How you name an item: `crate::models::User`, `std::collections::HashMap` |

A package can have one library crate plus any number of binaries (`src/main.rs`,
`src/bin/*.rs`, and examples).

## 2. Modules and privacy

**Everything is private by default.** An item is visible to its own module and
that module's children. Use `pub` to expose it.

```rust
mod shop {
    pub struct Product {
        pub name: String,     // public field
        price_cents: u32,     // private field: can't be set from outside `shop`
    }

    impl Product {
        pub fn new(name: &str, price_cents: u32) -> Self {   // public constructor
            Self { name: name.to_string(), price_cents }
        }
        pub fn price(&self) -> f64 {
            to_dollars(self.price_cents)
        }
    }

    fn to_dollars(cents: u32) -> f64 {   // private helper
        cents as f64 / 100.0
    }

    pub mod discounts {
        use super::Product;               // super = parent module
        pub fn half_off(p: &Product) -> f64 {
            p.price() / 2.0
        }
    }
}

use shop::Product;                         // bring a name into scope
use shop::discounts::half_off;
```

Private fields plus a public constructor is how Rust enforces **invariants**. For
example, nobody outside the module can build a `Product` with an unchecked price.

Path keywords:

- `crate::a::b`: absolute path from the crate root
- `super::x`: the parent module
- `self::x`: the current module

`use` tips:

```rust
use std::collections::{HashMap, HashSet};   // nested imports
use std::io::{self, Write};                 // `io` itself plus io::Write
use std::fmt::Result as FmtResult;          // rename to avoid clashes
pub use crate::shop::Product;               // re-export: part of YOUR public API
```

## 3. Modules in files: how real projects are laid out

Writing `mod name;` (with a semicolon) tells the compiler to look for the module in
a file:

```
src/
├── main.rs          ← contains: mod models; mod routes;
├── models.rs        ← module `models`
├── routes.rs        ← module `routes`, which can contain: mod admin;
└── routes/
    └── admin.rs     ← module `routes::admin`
```

```rust
// src/main.rs
mod models;          // loads src/models.rs
mod routes;          // loads src/routes.rs

fn main() {
    let u = models::User::new("yahia");
    routes::start();
}
```

**Important:** a file is **not** a module until some parent declares it with
`mod`. There's no automatic discovery. You'll use exactly this layout for the
service in chapter 10.

### Library + binary pattern

A common layout: logic lives in `src/lib.rs` (crate name `service_practice`), and
`src/main.rs` is a thin wrapper that calls into it with
`use service_practice::...`. That makes the logic testable and reusable.

## 4. External crates

```bash
cargo add rand              # adds `rand = "x.y"` to [dependencies]
cargo add serde --features derive
cargo add --dev pretty_assertions   # only for tests
```

Then use it by its crate name: `use rand::Rng;`. Crates are found on
https://crates.io, and their docs are on https://docs.rs.

Versioning: `serde = "1.0"` means "any **semver-compatible** version ≥ 1.0.0 and
< 2.0.0". `Cargo.lock` pins the exact version, so builds are reproducible.

## 5. Tests ⭐

Testing is built into the language and Cargo. No framework is needed.

### Unit tests: in the same file as the code

```rust
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]           // only compiled when running `cargo test`
mod tests {
    use super::*;      // tests are a child module, so they can see private items too

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 2), 4);
    }

    #[test]
    fn helpful_message() {
        let result = add(1, 1);
        assert!(result > 1, "expected > 1 but got {result}");
    }

    #[test]
    #[should_panic(expected = "divide by zero")]
    fn panics_on_zero() {
        divide(1, 0);
    }

    #[test]
    fn can_use_result() -> Result<(), String> {   // `?` works inside tests
        let n: i32 = "42".parse().map_err(|e| format!("{e}"))?;
        assert_eq!(n, 42);
        Ok(())
    }
}
```

The macros: `assert!(cond)`, `assert_eq!(left, right)`, `assert_ne!(left, right)`.
All of them accept an optional format message.

### Integration tests: in `tests/`

Files in `tests/*.rs` are compiled as separate crates that use your **library**'s
public API only, exactly as an outside user would.

### Doc tests

Code blocks inside `///` doc comments are **compiled and run** by `cargo test`, so
your documentation examples can't go stale:

```rust
/// Adds two numbers.
///
/// ```
/// assert_eq!(service_practice::add(1, 2), 3);
/// ```
pub fn add(a: i32, b: i32) -> i32 { a + b }
```

### Running tests

```bash
cargo test                       # everything
cargo test adds                  # only tests whose name contains "adds"
cargo test -- --nocapture        # show println! output from tests
cargo test --example 04_modules  # run the tests inside one example file
```

## 6. Type this: `examples/04_modules.rs`

```rust
// file: examples/04_modules.rs
mod bank {
    #[derive(Debug)]
    pub struct Account {
        owner: String,
        balance_cents: i64, // private: only `bank` can touch it directly
    }

    #[derive(Debug, PartialEq)]
    pub enum TxError {
        InsufficientFunds { needed: i64, available: i64 },
        InvalidAmount,
    }

    impl Account {
        pub fn open(owner: &str) -> Self {
            Self {
                owner: owner.to_string(),
                balance_cents: 0,
            }
        }

        pub fn owner(&self) -> &str {
            &self.owner
        }

        pub fn balance(&self) -> i64 {
            self.balance_cents
        }

        pub fn deposit(&mut self, cents: i64) -> Result<(), TxError> {
            validate(cents)?; // `?` returns the error early (chapter 05)
            self.balance_cents += cents;
            Ok(())
        }

        pub fn withdraw(&mut self, cents: i64) -> Result<(), TxError> {
            validate(cents)?;
            if cents > self.balance_cents {
                return Err(TxError::InsufficientFunds {
                    needed: cents,
                    available: self.balance_cents,
                });
            }
            self.balance_cents -= cents;
            Ok(())
        }
    }

    fn validate(cents: i64) -> Result<(), TxError> {
        if cents <= 0 {
            Err(TxError::InvalidAmount)
        } else {
            Ok(())
        }
    }

    #[cfg(test)]
    mod tests {
        use super::*;

        #[test]
        fn deposit_increases_balance() {
            let mut acc = Account::open("test");
            acc.deposit(500).unwrap();
            assert_eq!(acc.balance(), 500);
        }

        #[test]
        fn cannot_overdraw() {
            let mut acc = Account::open("test");
            acc.deposit(100).unwrap();
            assert_eq!(
                acc.withdraw(250),
                Err(TxError::InsufficientFunds { needed: 250, available: 100 })
            );
            assert_eq!(acc.balance(), 100); // unchanged
        }

        #[test]
        fn rejects_negative_amounts() {
            let mut acc = Account::open("test");
            assert_eq!(acc.deposit(-5), Err(TxError::InvalidAmount));
        }

        #[test]
        fn private_helper_is_testable() {
            assert!(validate(1).is_ok()); // child modules can see private items
        }
    }
}

use bank::{Account, TxError};

fn main() {
    let mut acc = Account::open("yahia");
    acc.deposit(10_000).unwrap();

    match acc.withdraw(25_000) {
        Ok(()) => println!("withdrawn"),
        Err(TxError::InsufficientFunds { needed, available }) => {
            println!("need {needed}, only {available} available")
        }
        Err(e) => println!("other error: {e:?}"),
    }

    // acc.balance_cents = 1_000_000;  // ❌ E0616: field is private
    println!("{} has {} cents", acc.owner(), acc.balance());
}
```

```bash
cargo run --example 04_modules
cargo test --example 04_modules
```

## ✅ Exercises

1. Add a `transfer(from: &mut Account, to: &mut Account, cents: i64) -> Result<(), TxError>`
   function inside `bank`, with a test for it. (Can you call `transfer(&mut a, &mut a, 5)`?
   Why not? Hint: the borrowing rules.)
2. Make one test fail on purpose and read the output of `cargo test`.
3. Practice file modules for real: move `bank` out of the example into a library.
   Create `src/lib.rs` containing `pub mod bank;`, move the module body to
   `src/bank.rs`, and in the example write `use service_practice::bank::Account;`.
   (Undo this afterwards if you want a clean slate for chapter 10, or keep it.
   Both work.)

<details>
<summary>Solution for 1</summary>

```rust
// inside `mod bank`
pub fn transfer(from: &mut Account, to: &mut Account, cents: i64) -> Result<(), TxError> {
    from.withdraw(cents)?;
    to.deposit(cents)?;
    Ok(())
}

// inside `mod tests`
#[test]
fn transfer_moves_money() {
    let mut a = Account::open("a");
    let mut b = Account::open("b");
    a.deposit(300).unwrap();
    transfer(&mut a, &mut b, 100).unwrap();
    assert_eq!((a.balance(), b.balance()), (200, 100));
}
```

`transfer(&mut a, &mut a, 5)` is rejected with E0499 ("cannot borrow `a` as mutable
more than once at a time"). Two `&mut` to the same value would break the "one
writer" rule.

</details>

Next → [05: Collections and error handling](05-collections-errors.md)
