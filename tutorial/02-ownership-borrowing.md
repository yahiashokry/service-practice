# 02: Ownership, borrowing and slices ⭐

> Official reading: [The Book, ch. 4 "Understanding Ownership"](https://doc.rust-lang.org/book/ch04-00-understanding-ownership.html)
> (read all three sections, twice if needed) · [Rust by Example: Scoping rules](https://doc.rust-lang.org/rust-by-example/scope.html)

This is **the** chapter. Ownership is what makes Rust different from every
language you've used. It's how Rust gets memory safety **without a garbage
collector**. Take your time here.

## 1. The problem ownership solves

Programs put data in two places:

- **Stack**: fast, fixed-size, last-in-first-out. Local `i32`s, `bool`s, fixed arrays.
  Freed automatically when the function returns.
- **Heap**: for data whose size is unknown at compile time or can change (`String`,
  `Vec`). You ask the allocator for memory, get back a **pointer**, and someone must
  **free** it exactly once.

Other languages handle "free it exactly once" in two ways:

| Approach | Languages | Cost |
|----------|-----------|------|
| You free manually | C, C++ | Forget → leak. Twice → crash or security bug. Too early → use-after-free. |
| Garbage collector | Java, Go, Python, JS | Runtime overhead, pauses, less control |
| **Ownership** | **Rust** | The compiler inserts the `free` for you, checked at compile time. No runtime cost. |

## 2. The three ownership rules

> 1. Each value in Rust has an **owner** (a variable).
> 2. There can only be **one owner at a time**.
> 3. When the owner goes **out of scope**, the value is **dropped** (freed).

```rust
{
    let s = String::from("hello"); // s owns the heap memory
    // use s
}                                  // s goes out of scope → memory freed (`drop` is called)
```

## 3. Move: assignment transfers ownership

A `String` is three values on the stack: a pointer to the heap, a length, and a
capacity.

```
   s1 (stack)                 heap
 ┌──────────┐              ┌───┬───┬───┬───┬───┐
 │ ptr  ────┼─────────────►│ h │ e │ l │ l │ o │
 │ len  = 5 │              └───┴───┴───┴───┴───┘
 │ cap  = 5 │
 └──────────┘
```

```rust
let s1 = String::from("hello");
let s2 = s1;              // the pointer/len/cap are COPIED, the heap data is NOT
println!("{s1}");         // ❌ error[E0382]: borrow of moved value: `s1`
```

If both `s1` and `s2` owned the heap data, both would free it at the end of the
scope: a **double free**. So Rust says `s1` is **moved** into `s2`, and `s1` can't
be used anymore. Only one owner remains (rule 2).

Read the full error with `cargo check`. It shows where the move happened, and often
suggests a fix:

```
help: consider cloning the value if the performance cost is acceptable
  |     let s2 = s1.clone();
```

### Clone: an explicit deep copy

```rust
let s1 = String::from("hello");
let s2 = s1.clone();      // copies the heap data too; now two independent strings
println!("{s1} {s2}");    // ✅
```

`.clone()` is visible in the code, so you can *see* where expensive copies happen.

### Copy types: stack-only values are copied, not moved

```rust
let x = 5;
let y = x;                // i32 is `Copy`: a cheap bit-copy
println!("{x} {y}");      // ✅ both valid
```

Types that implement `Copy`: all integers, floats, `bool`, `char`, and tuples or
arrays made only of `Copy` types. Anything that owns heap memory (`String`, `Vec`,
`Box`) is **not** `Copy`.

## 4. Ownership and functions

Passing a value to a function works exactly like assignment: it **moves** or
**copies**.

```rust
fn takes_ownership(s: String) {   // s comes into scope
    println!("{s}");
}                                  // s is dropped here; memory freed

fn main() {
    let s = String::from("hi");
    takes_ownership(s);            // s is MOVED into the function
    // println!("{s}");            // ❌ s is no longer valid here
}
```

Returning a value moves ownership *out*:

```rust
fn make() -> String {
    String::from("new")            // ownership moves to the caller
}
```

Passing ownership in and returning it back every time would be tiring. That's why
references exist.

## 5. References and borrowing

A **reference** `&T` lets you *use* a value without *owning* it. Creating one is
called **borrowing**.

```rust
fn length(s: &String) -> usize {  // s is a reference: we borrow, we don't own
    s.len()
}                                  // s goes out of scope, but nothing is dropped
                                   // (it never owned anything)

let s1 = String::from("hello");
let len = length(&s1);             // `&s1` creates a reference
println!("{s1} has length {len}"); // ✅ s1 still valid
```

```
   s (reference)      s1 (owner)                heap
 ┌──────────┐       ┌──────────┐            ┌───┬───┬───┬───┬───┐
 │ ptr  ────┼──────►│ ptr  ────┼───────────►│ h │ e │ l │ l │ o │
 └──────────┘       │ len  = 5 │            └───┴───┴───┴───┴───┘
                    │ cap  = 5 │
                    └──────────┘
```

### Mutable references

```rust
fn add_world(s: &mut String) {
    s.push_str(", world");
}

let mut s = String::from("hello");
add_world(&mut s);                 // both the variable and the borrow need `mut`
```

## 6. The borrowing rules ⭐⭐

> At any given time, you can have **EITHER**
> - **one** mutable reference (`&mut T`), **OR**
> - **any number** of immutable references (`&T`).
>
> And references must always be **valid** (never dangle).

Think of it as a **read-write lock checked at compile time**: many readers, or one
writer, never both.

```rust
let mut s = String::from("hello");

let r1 = &s;
let r2 = &s;          // ✅ many readers are fine
let r3 = &mut s;      // ❌ error[E0502]: cannot borrow `s` as mutable
                      //    because it is also borrowed as immutable
println!("{r1} {r2}");
```

**Why this rule?** It prevents **data races** and a whole class of bugs such as
"I'm iterating a list while something else modifies it". Here's the classic
example. It compiles in many languages and crashes or misbehaves at runtime. In
Rust it doesn't compile:

```rust
let mut v = vec![1, 2, 3];
let first = &v[0];    // immutable borrow of v
v.push(4);            // ❌ mutable borrow, and push may REALLOCATE the vec,
                      //    which would leave `first` pointing at freed memory!
println!("{first}");
```

### Borrows end at their last use (Non-Lexical Lifetimes)

A borrow lasts until the last place it's *used*, not until the end of the block:

```rust
let mut s = String::from("hello");
let r1 = &s;
let r2 = &s;
println!("{r1} {r2}");   // last use of r1 and r2: their borrows end HERE
let r3 = &mut s;          // ✅ fine now
r3.push('!');
```

### No dangling references

```rust
fn dangle() -> &String {      // ❌ error[E0106]: missing lifetime specifier
    let s = String::from("hi");
    &s                         // s is dropped at the end of this function,
}                              // so the reference would point to freed memory
```

The fix: return the `String` itself (move ownership out). Chapter 06 covers
**lifetimes**, the way the compiler tracks how long references are valid.

## 7. Slices: references to part of a collection

```rust
let s = String::from("hello world");
let hello: &str = &s[0..5];    // a "string slice": pointer + length into s
let world: &str = &s[6..];
let whole: &str = &s[..];

let a = [1, 2, 3, 4, 5];
let part: &[i32] = &a[1..3];   // [2, 3]
```

### `String` vs `&str`: the question every beginner asks

| | `String` | `&str` |
|---|---|---|
| Owns its data? | Yes (heap-allocated, growable) | No (a borrowed view) |
| Can grow (`push_str`)? | Yes, if `mut` | No |
| String literals `"hi"` | — | `&'static str` (baked into the binary) |
| Use as function **parameter** | Rarely | **Usually.** It accepts both. |
| Use as struct **field** | **Usually** | Only with lifetimes (advanced) |

```rust
fn first_word(s: &str) -> &str {           // takes &str: the flexible choice
    s.split(' ').next().unwrap_or("")
}

let owned = String::from("hello world");
first_word(&owned);       // &String auto-converts to &str ("deref coercion")
first_word("literal");    // &'static str works too
```

**Rule of thumb:** take `&str` / `&[T]` as parameters, and store `String` / `Vec<T>`
in structs.

## 8. Type this: `examples/02_ownership.rs`

Type it, run it, then **uncomment each ❌ line one at a time** and read the error.

```rust
// file: examples/02_ownership.rs
fn calculate_length(s: &str) -> usize {
    s.len()
}

fn append_exclaim(s: &mut String) {
    s.push('!');
}

fn take_and_give_back(mut s: String) -> String {
    s.push_str(" (processed)");
    s
}

fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

fn main() {
    // --- move ---
    let a = String::from("hello");
    let b = a; // a moved into b
    // println!("{a}");        // ❌ E0382: borrow of moved value
    println!("b = {b}");

    // --- clone ---
    let c = b.clone();
    println!("b = {b}, c = {c}");

    // --- copy ---
    let n = 42;
    let m = n;
    println!("n = {n}, m = {m}");

    // --- borrowing ---
    let len = calculate_length(&c);
    println!("'{c}' has {len} bytes");

    // --- mutable borrowing ---
    let mut d = String::from("wow");
    append_exclaim(&mut d);
    println!("d = {d}");

    // --- the borrowing rule ---
    let r1 = &d;
    let r2 = &d;
    // let r3 = &mut d;        // ❌ E0502 (if r1/r2 are used afterwards)
    println!("{r1} and {r2}");
    let r3 = &mut d; // ✅ r1 and r2 are no longer used
    r3.push('?');
    println!("d = {d}");

    // --- ownership in and out ---
    let e = take_and_give_back(d);
    // println!("{d}");        // ❌ d was moved into the function
    println!("e = {e}");

    // --- slices ---
    let sentence = String::from("ownership is the key");
    let w = first_word(&sentence);
    println!("first word: {w}");

    // let mut sentence2 = String::from("hello there");
    // let w2 = first_word(&sentence2);
    // sentence2.clear();       // ❌ E0502: can't mutate while w2 borrows it
    // println!("{w2}");

    // --- drop order ---
    {
        let _inner = String::from("I live in a small scope");
        println!("inner scope running");
    } // _inner dropped here
    println!("after inner scope");
}
```

```bash
cargo run --example 02_ownership
```

## 9. Summary: decision guide when writing a function

| You need to... | Parameter type |
|---|---|
| just read the data | `&T` (or `&str` / `&[T]`) |
| modify the caller's data | `&mut T` |
| keep or store the data, or consume it | `T` (take ownership) |

If you're fighting the borrow checker, the usual fixes are (in order of preference):
restructure so borrows don't overlap, borrow for a shorter time, `.clone()` (fine
while learning, and often fine in general), and later shared ownership
(`Rc`/`Arc`, chapter 08).

## ✅ Exercises

Create `examples/02_exercises.rs`:

1. Write `fn longest_word(text: &str) -> &str` returning the longest
   whitespace-separated word. Hint: `text.split_whitespace()`.
2. Write `fn to_shout(s: &mut String)` that uppercases the string in place and adds
   `"!!!"`. Hint: `*s = s.to_uppercase();`
3. Write `fn consume(v: Vec<i32>) -> i32` that returns the sum. Call it, then try to
   use the vec afterwards and read the error. Fix it in **two** ways: (a) change the
   parameter to `&[i32]`, (b) clone at the call site.
4. Explain to yourself, in words, why `let first = &v[0]; v.push(4); println!("{first}")`
   is rejected.

<details>
<summary>Solution</summary>

```rust
// file: examples/02_exercises.rs
fn longest_word(text: &str) -> &str {
    let mut best = "";
    for word in text.split_whitespace() {
        if word.len() > best.len() {
            best = word;
        }
    }
    best
}

fn to_shout(s: &mut String) {
    *s = s.to_uppercase(); // `*s` = "the String that s points to"
    s.push_str("!!!");
}

fn sum_slice(v: &[i32]) -> i32 {
    let mut total = 0;
    for x in v {
        total += x;
    }
    total
}

fn consume(v: Vec<i32>) -> i32 {
    v.iter().sum()
}

fn main() {
    println!("{}", longest_word("rust makes memory safety practical"));

    let mut s = String::from("hello");
    to_shout(&mut s);
    println!("{s}");

    let v = vec![1, 2, 3];
    println!("{}", sum_slice(&v)); // (a) borrow: v still usable
    println!("{}", consume(v.clone())); // (b) clone: give away a copy
    println!("still have {v:?}");

    // 4. push() may reallocate the Vec's buffer to a new heap location;
    //    `first` would then point at freed memory. The borrow rules make
    //    this impossible: no mutation while an immutable borrow is alive.
}
```

</details>

Next → [03: Structs, enums and pattern matching](03-structs-enums-matching.md)
