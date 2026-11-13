# 05: Collections and error handling

> Official reading: [The Book, ch. 8 "Common Collections"](https://doc.rust-lang.org/book/ch08-00-common-collections.html),
> [ch. 9 "Error Handling"](https://doc.rust-lang.org/book/ch09-00-error-handling.html)
> · std docs: [`Vec`](https://doc.rust-lang.org/std/vec/struct.Vec.html), [`String`](https://doc.rust-lang.org/std/string/struct.String.html),
> [`HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html), [`Result`](https://doc.rust-lang.org/std/result/enum.Result.html)

## Part A: Collections

### `Vec<T>`: a growable list

```rust
let mut v: Vec<i32> = Vec::new();
let mut v = vec![1, 2, 3];      // macro shortcut
v.push(4);
let last = v.pop();             // Option<i32>: the vec might be empty

let third: &i32 = &v[2];        // panics if out of bounds
let third: Option<&i32> = v.get(2);  // safe: None if out of bounds

for x in &v { }                 // borrow each element (&i32)
for x in &mut v { *x *= 2; }    // mutate each element (&mut i32); `*` dereferences
for x in v { }                  // CONSUMES the vec: v is moved into the loop
```

These three `for` forms are ownership again: borrow, borrow mutably, or take
ownership.

### `String`: UTF-8 text

```rust
let mut s = String::new();
let s = String::from("hello");
let s = "hello".to_string();
let mut s = String::from("foo");
s.push_str("bar");
s.push('!');
let combined = format!("{s}-{}", "baz");   // format! never takes ownership

for c in "héllo".chars() { }    // Unicode scalar values
for b in "héllo".bytes() { }    // raw bytes
"héllo".len()                   // 6: len is in BYTES, not characters!
```

Why does `s[0]` not compile? Strings are UTF-8, and a "character" can be 1 to 4
bytes. Rust won't let you index into the middle of one by accident. Use `.chars()`,
or slice with a byte range you know is valid.

### `HashMap<K, V>`

```rust
use std::collections::HashMap;

let mut scores: HashMap<String, u32> = HashMap::new();
scores.insert("blue".to_string(), 10);

let blue: Option<&u32> = scores.get("blue");

// The Entry API: "insert if missing, then modify". Very idiomatic.
let text = "the cat the dog the end";
let mut counts: HashMap<&str, usize> = HashMap::new();
for word in text.split_whitespace() {
    *counts.entry(word).or_insert(0) += 1;
}

for (word, n) in &counts { println!("{word}: {n}"); }  // order is NOT guaranteed
```

Others in `std::collections`: `HashSet`, `BTreeMap` (sorted keys), `VecDeque`
(queue), `BinaryHeap`.

## Part B: Error handling ⭐

Rust has **no exceptions**. Errors come in two kinds:

| Kind | Mechanism | When |
|------|-----------|------|
| **Unrecoverable** | `panic!` (crashes the thread with a message) | Bugs: a broken invariant, "this can't happen" |
| **Recoverable** | `Result<T, E>`, a normal return value | Expected failures: file missing, bad input, network down |

### `Result<T, E>`

```rust
enum Result<T, E> {   // from std, always in scope
    Ok(T),
    Err(E),
}
```

Because errors are just return values, a function's signature **tells you it can
fail**, and the compiler warns you if you ignore a `Result` (`#[must_use]`).

```rust
use std::fs;

match fs::read_to_string("config.txt") {
    Ok(contents) => println!("{contents}"),
    Err(e) => eprintln!("could not read config: {e}"),
}
```

### The `?` operator: propagate errors concisely

```rust
use std::num::ParseIntError;

fn parse_sum(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let x: i32 = a.parse()?;   // if Err, return it from parse_sum right now
    let y: i32 = b.parse()?;   // if Ok, unwrap the value and continue
    Ok(x + y)
}
```

`expr?` is roughly shorthand for:

```rust
match expr {
    Ok(v) => v,
    Err(e) => return Err(From::from(e)),   // note the From conversion
}
```

The `From::from` call means `?` can **convert** error types automatically, as long as
a `From` implementation exists. `?` also works on `Option` inside a function that
returns `Option`.

### `unwrap`, `expect` and friends

```rust
let n: i32 = "42".parse().unwrap();                  // panic on Err: fine in prototypes and tests
let n: i32 = "42".parse().expect("valid number");    // panic with YOUR message: better
let n: i32 = "x".parse().unwrap_or(0);               // fallback value
let n: i32 = "x".parse().unwrap_or_default();        // 0 for numbers, "" for String...
let r: Result<i32, String> = "x".parse::<i32>().map_err(|e| e.to_string()); // convert the error
let o: Option<i32> = "x".parse().ok();               // Result → Option
```

**Guideline:** in application code, prefer `?` and handle errors at the top level.
Use `expect("why this can't fail")` when you know better than the compiler.

### Custom error types

Real programs combine many error sources. The idiomatic pattern is an enum, plus
`Display`, `std::error::Error`, and `From` conversions so that `?` works:

```rust
use std::fmt;

#[derive(Debug)]
enum ConfigError {
    Io(std::io::Error),
    BadPort(String),
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::Io(e) => write!(f, "io error: {e}"),
            ConfigError::BadPort(p) => write!(f, "invalid port: {p}"),
        }
    }
}

impl std::error::Error for ConfigError {}

impl From<std::io::Error> for ConfigError {    // lets `?` convert io::Error → ConfigError
    fn from(e: std::io::Error) -> Self { ConfigError::Io(e) }
}
```

In practice, people use two crates to remove this boilerplate:

- **`thiserror`**: derive macros for custom error enums, used in **libraries** and
  in the service in chapter 10.
- **`anyhow`**: a single "any error" type plus context, convenient in
  **applications** and scripts.

`main` can return a `Result` too:
`fn main() -> Result<(), Box<dyn std::error::Error>> { ...; Ok(()) }`

## Type this: `examples/05_inventory.rs`

```rust
// file: examples/05_inventory.rs
use std::collections::HashMap;
use std::fmt;

#[derive(Debug)]
enum InvError {
    Parse { line: usize, reason: String },
    UnknownItem(String),
    OutOfStock { item: String, have: u32, want: u32 },
}

impl fmt::Display for InvError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            InvError::Parse { line, reason } => write!(f, "line {line}: {reason}"),
            InvError::UnknownItem(name) => write!(f, "unknown item '{name}'"),
            InvError::OutOfStock { item, have, want } => {
                write!(f, "'{item}': wanted {want}, only {have} left")
            }
        }
    }
}

impl std::error::Error for InvError {}

struct Inventory {
    stock: HashMap<String, u32>,
}

impl Inventory {
    /// Parses lines like "apple=5". Fails on the first bad line.
    fn parse(input: &str) -> Result<Self, InvError> {
        let mut stock = HashMap::new();
        for (i, raw) in input.lines().enumerate() {
            let line = raw.trim();
            if line.is_empty() || line.starts_with('#') {
                continue;
            }
            let Some((name, qty)) = line.split_once('=') else {
                return Err(InvError::Parse {
                    line: i + 1,
                    reason: format!("expected name=qty, got '{line}'"),
                });
            };
            let qty: u32 = qty.trim().parse().map_err(|e| InvError::Parse {
                line: i + 1,
                reason: format!("bad quantity: {e}"),
            })?;
            *stock.entry(name.trim().to_string()).or_insert(0) += qty;
        }
        Ok(Self { stock })
    }

    fn take(&mut self, item: &str, want: u32) -> Result<u32, InvError> {
        let have = self
            .stock
            .get_mut(item)
            .ok_or_else(|| InvError::UnknownItem(item.to_string()))?;

        if *have < want {
            return Err(InvError::OutOfStock {
                item: item.to_string(),
                have: *have,
                want,
            });
        }
        *have -= want;
        Ok(*have)
    }
}

fn run() -> Result<(), Box<dyn std::error::Error>> {
    let data = "
        # fruit stock
        apple = 5
        banana = 12
        apple = 3
    ";
    let mut inv = Inventory::parse(data)?;

    let left = inv.take("apple", 6)?;
    println!("took 6 apples, {left} left");

    for (item, qty) in [("banana", 20), ("kiwi", 1), ("banana", 2)] {
        match inv.take(item, qty) {
            Ok(left) => println!("took {qty} {item}, {left} left"),
            Err(e) => println!("error: {e}"),
        }
    }

    let bad = Inventory::parse("pear = lots");
    if let Err(e) = bad {
        println!("parse failed as expected: {e}");
    }

    Ok(())
}

fn main() {
    if let Err(e) = run() {
        eprintln!("fatal: {e}");
        std::process::exit(1);
    }
}
```

```bash
cargo run --example 05_inventory
```

## ✅ Exercises

Create `examples/05_exercises.rs`:

1. Word frequency: given a paragraph, print the 3 most common words (lowercased,
   punctuation stripped). Hints: `HashMap`, `.entry()`, collect the pairs into a
   `Vec<(&str, usize)>`, then `.sort_by(|a, b| b.1.cmp(&a.1))`.
2. Write `fn parse_kv(s: &str) -> Result<(String, i64), String>` for input like
   `"age=30"`. Return descriptive errors for a missing `=` and for a non-number value.
3. Write `fn average(nums: &[f64]) -> Option<f64>` (None for an empty slice) and
   `fn first_even(nums: &[i32]) -> Option<i32>`.

<details>
<summary>Solution</summary>

```rust
// file: examples/05_exercises.rs
use std::collections::HashMap;

fn top_words(text: &str, n: usize) -> Vec<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for raw in text.split_whitespace() {
        let word: String = raw
            .chars()
            .filter(|c| c.is_alphanumeric())
            .collect::<String>()
            .to_lowercase();
        if !word.is_empty() {
            *counts.entry(word).or_insert(0) += 1;
        }
    }
    let mut pairs: Vec<(String, usize)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    pairs.truncate(n);
    pairs
}

fn parse_kv(s: &str) -> Result<(String, i64), String> {
    let (k, v) = s
        .split_once('=')
        .ok_or_else(|| format!("missing '=' in '{s}'"))?;
    let v: i64 = v
        .trim()
        .parse()
        .map_err(|e| format!("value '{v}' is not a number: {e}"))?;
    Ok((k.trim().to_string(), v))
}

fn average(nums: &[f64]) -> Option<f64> {
    if nums.is_empty() {
        None
    } else {
        Some(nums.iter().sum::<f64>() / nums.len() as f64)
    }
}

fn first_even(nums: &[i32]) -> Option<i32> {
    // loop version: for &n in nums { if n % 2 == 0 { return Some(n); } } None
    // iterator version (preview of ch. 07):
    nums.iter().copied().find(|n| n % 2 == 0)
}

fn main() {
    let text = "The cat and the dog. The dog barked; the cat ran!";
    println!("{:?}", top_words(text, 3));

    println!("{:?}", parse_kv("age=30"));
    println!("{:?}", parse_kv("age30"));
    println!("{:?}", parse_kv("age=thirty"));

    println!("{:?} {:?}", average(&[1.0, 2.0, 4.5]), average(&[]));
    println!("{:?} {:?}", first_even(&[1, 3, 8, 10]), first_even(&[1, 3]));
}
```

</details>

Next → [06: Generics, traits and lifetimes](06-generics-traits-lifetimes.md)
