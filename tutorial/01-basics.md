# 01: Basics: variables, types, functions, control flow

> Official reading: [The Book, ch. 3 "Common Programming Concepts"](https://doc.rust-lang.org/book/ch03-00-common-programming-concepts.html)
> · [Rust by Example: Primitives](https://doc.rust-lang.org/rust-by-example/primitives.html)

## 1. Variables are immutable by default

```rust
let x = 5;
x = 6;        // ❌ error[E0384]: cannot assign twice to immutable variable `x`
```

```rust
let mut x = 5; // `mut` opts in to mutation
x = 6;         // ✅
```

**Why?** When you read code, an immutable binding can't change behind your back.
Mutability is visible in the source (`mut`), so you can see where state changes.
This becomes important for the borrowing rules in chapter 02.

### Shadowing

You can declare a *new* variable with the same name. It can even have a different type:

```rust
let input = "42";              // &str
let input: i32 = input.parse().unwrap(); // a NEW variable named input, type i32
```

This is different from `mut`. Shadowing creates a new binding, and the old one
still exists but can no longer be reached by that name. It's commonly used for
"transform this value and keep the name".

### Constants

```rust
const MAX_POINTS: u32 = 100_000;   // type annotation is REQUIRED, evaluated at compile time
```

## 2. Scalar types

| Kind | Types | Notes |
|------|-------|-------|
| Signed integers | `i8 i16 i32 i64 i128 isize` | `i32` is the default |
| Unsigned integers | `u8 u16 u32 u64 u128 usize` | `usize` = pointer size, used for indexing |
| Floats | `f32 f64` | `f64` is the default |
| Boolean | `bool` | `true` / `false` |
| Character | `char` | 4 bytes, one Unicode scalar value: `'z'`, `'😻'` |

Rust **infers types** but never **converts them implicitly**:

```rust
let a: i32 = 10;
let b: i64 = 20;
let c = a + b;            // ❌ mismatched types
let c = a as i64 + b;     // ✅ explicit cast with `as`
let d = i64::from(a) + b; // ✅ lossless conversion (preferred when available)
```

**Integer overflow:** in debug builds, `255u8 + 1` **panics** (crashes with a
message). In release builds it wraps around. When you want specific behavior, say
so explicitly: `x.wrapping_add(1)`, `x.checked_add(1)` (returns `Option`), or
`x.saturating_add(1)`.

## 3. Compound types: tuples and arrays

```rust
let tup: (i32, f64, char) = (500, 6.4, 'x');
let (a, b, c) = tup;        // destructuring
let first = tup.0;          // access by index

let arr = [1, 2, 3, 4, 5];  // fixed length, type [i32; 5], lives on the stack
let zeros = [0; 10];        // ten zeros
let third = arr[2];
```

Out-of-bounds indexing (`arr[10]`) **panics** at runtime instead of reading random
memory. That's memory safety. For a growable list you'll use `Vec<T>` (chapter 05).

The unit type `()` is the empty tuple. It's what functions "return" when they
return nothing.

## 4. Functions, statements, expressions ⭐

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b        // no semicolon → this expression is the return value
}
```

This is one of the most important syntax ideas in Rust:

- An **expression** produces a value: `5`, `a + b`, `if ... { } else { }`, a `{ block }`.
- A **statement** performs an action and produces no value: `let x = 5;`
- Adding `;` to an expression turns it into a statement, and its value is thrown away.

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b;   // ❌ error: mismatched types: expected `i32`, found `()`
}            //    (the compiler even suggests: "remove this semicolon")
```

Blocks are expressions too:

```rust
let y = {
    let x = 3;
    x + 1          // the block's value is 4
};
```

`return` exists for early returns. The last expression is the usual way to return
a value.

## 5. Control flow

`if` is an expression, so it returns a value (Rust has no ternary operator because
it doesn't need one):

```rust
let n = 7;
let kind = if n % 2 == 0 { "even" } else { "odd" };
```

The condition **must be a `bool`**. `if n { }` doesn't compile, because Rust has no
"truthy" values.

Loops:

```rust
// loop: runs forever until `break`; can return a value through break
let mut counter = 0;
let result = loop {
    counter += 1;
    if counter == 10 {
        break counter * 2;   // loop evaluates to 20
    }
};

// while
let mut n = 3;
while n > 0 {
    n -= 1;
}

// for: iterates over anything iterable. This is the loop you'll use most.
for i in 0..5 { }          // 0,1,2,3,4   (exclusive range)
for i in 0..=5 { }         // 0..5 inclusive
for x in [10, 20, 30] { }
for (i, x) in ["a", "b"].iter().enumerate() { }

// labeled loops: break out of an outer loop
'outer: for i in 0..10 {
    for j in 0..10 {
        if i * j > 20 { break 'outer; }
    }
}
```

## 6. Type this: `examples/01_basics.rs`

```rust
// file: examples/01_basics.rs
const FREEZING_F: f64 = 32.0;

fn fahrenheit_to_celsius(f: f64) -> f64 {
    (f - FREEZING_F) * 5.0 / 9.0
}

fn classify(n: i32) -> &'static str {
    if n < 0 {
        "negative"
    } else if n == 0 {
        "zero"
    } else {
        "positive"
    }
}

fn main() {
    // mutability and shadowing
    let mut total = 0;
    for i in 1..=10 {
        total += i;
    }
    println!("sum 1..=10 = {total}");

    let total = total as f64 / 10.0; // shadowed: now an f64
    println!("average = {total}");

    // tuples
    let point = (3, -7);
    let (x, y) = point;
    println!("x={x} is {}, y={y} is {}", classify(x), classify(y));

    // arrays and for loops
    let temps_f = [32.0, 50.0, 98.6, 212.0];
    for (i, f) in temps_f.iter().enumerate() {
        println!("#{i}: {f}°F = {:.1}°C", fahrenheit_to_celsius(*f));
    }

    // loop with a value
    let mut attempts = 0;
    let found = loop {
        attempts += 1;
        if attempts * attempts > 50 {
            break attempts;
        }
    };
    println!("first n with n² > 50 is {found}");

    // overflow handling done explicitly
    let big: u8 = 250;
    println!("checked:    {:?}", big.checked_add(10)); // None
    println!("wrapping:   {}", big.wrapping_add(10));  // 4
    println!("saturating: {}", big.saturating_add(10)); // 255
}
```

```bash
cargo run --example 01_basics
```

`{:.1}` means "one decimal place". `{:?}` is Debug formatting. You'll see both
everywhere. Full syntax: [`std::fmt`](https://doc.rust-lang.org/std/fmt/index.html).

## ✅ Exercises

Create `examples/01_exercises.rs` and write:

1. `fn fib(n: u32) -> u64` returning the n-th Fibonacci number, using a loop (no recursion).
2. FizzBuzz for 1..=30 using `for` and `if`/`else if`. Bonus: build the output with a `match` on `(n % 3, n % 5)`.
3. `fn is_prime(n: u32) -> bool`.
4. Try `let x: u8 = 256;` and read the compiler error.

<details>
<summary>Solution</summary>

```rust
// file: examples/01_exercises.rs
fn fib(n: u32) -> u64 {
    let (mut a, mut b) = (0u64, 1u64);
    for _ in 0..n {
        let next = a + b;
        a = b;
        b = next;
    }
    a
}

fn is_prime(n: u32) -> bool {
    if n < 2 {
        return false;
    }
    let mut d = 2;
    while d * d <= n {
        if n.is_multiple_of(d) {
            return false;
        }
        d += 1;
    }
    true
}

fn main() {
    for n in 0..10 {
        print!("{} ", fib(n));
    }
    println!();

    for n in 1..=30 {
        let out = match (n % 3, n % 5) {
            (0, 0) => "FizzBuzz".to_string(),
            (0, _) => "Fizz".to_string(),
            (_, 0) => "Buzz".to_string(),
            _ => n.to_string(),
        };
        println!("{out}");
    }

    let primes: Vec<u32> = (1..50).filter(|&n| is_prime(n)).collect();
    println!("{primes:?}");
}
```

</details>

Next → [02: Ownership and borrowing](02-ownership-borrowing.md)
