# 06: Generics, traits and lifetimes

> Official reading: [The Book, ch. 10 "Generic Types, Traits, and Lifetimes"](https://doc.rust-lang.org/book/ch10-00-generics.html),
> [ch. 18 "OOP features"](https://doc.rust-lang.org/book/ch18-00-oop.html) (trait objects)
> · [Rust by Example: Traits](https://doc.rust-lang.org/rust-by-example/trait.html)

These three topics are how Rust code becomes **reusable** while staying fast and
safe. In chapter 10 you'll see them everywhere in axum (`impl IntoResponse`,
`State<T>`, `Json<T>`).

## 1. Generics: code that works for many types

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

largest(&[1, 5, 3]);         // T = i32
largest(&['a', 'z', 'q']);   // T = char
```

`T: PartialOrd` is a **trait bound**: "T can be any type, as long as its values can
be compared with `>`". Without the bound, `item > largest` wouldn't compile,
because the compiler can't assume every type supports comparison.

Generic structs and enums:

```rust
struct Point<T> { x: T, y: T }

impl<T> Point<T> {
    fn x(&self) -> &T { &self.x }
}

impl Point<f64> {                    // methods only for Point<f64>
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

// You already know two generic enums:  Option<T>  and  Result<T, E>
```

**Zero cost:** generics are **monomorphized**. The compiler generates a separate
copy of the code for each concrete type used, so there's no runtime overhead
compared to writing it by hand.

## 2. Traits: shared behavior (like interfaces)

```rust
trait Summary {
    fn summarize_author(&self) -> String;          // required method

    fn summarize(&self) -> String {                // default method
        format!("(Read more from {}...)", self.summarize_author())
    }
}

struct Tweet { username: String, content: String }

impl Summary for Tweet {
    fn summarize_author(&self) -> String {
        format!("@{}", self.username)
    }
}
```

### Using traits as parameters: three equivalent spellings

```rust
fn notify(item: &impl Summary) { }                  // impl Trait: short form

fn notify<T: Summary>(item: &T) { }                 // trait bound: the full form

fn notify<T>(item: &T) where T: Summary + Clone { } // where clause: for long bounds
```

Returning `impl Trait` means "I return some concrete type that implements this trait,
and I'm not telling you which one". It's used a lot with iterators and closures.

```rust
fn make_counter() -> impl Iterator<Item = u32> {
    (1..=3).map(|x| x * 10)
}
```

### Static dispatch vs dynamic dispatch: `dyn Trait`

Generics require one concrete type per call site. To store **different types in one
collection**, use **trait objects**:

```rust
let items: Vec<Box<dyn Summary>> = vec![
    Box::new(Tweet { /* ... */ }),
    Box::new(Article { /* ... */ }),
];
for i in &items {
    println!("{}", i.summarize());   // the method is looked up at runtime (vtable)
}
```

| | Generics / `impl Trait` | `dyn Trait` |
|---|---|---|
| Dispatch | Static (compile time) | Dynamic (runtime vtable) |
| Speed | Fastest, can inline | Small indirection cost |
| Mixed types in one `Vec`? | No | Yes |

### The standard traits you'll use constantly

| Trait | Gives you | Usually |
|------|-----------|---------|
| `Debug` | `{:?}` formatting | `#[derive(Debug)]` on almost everything |
| `Clone` | `.clone()` | derive |
| `Copy` | implicit bit-copy instead of move | derive, for small plain-data types |
| `PartialEq`, `Eq` | `==`, `!=` | derive |
| `PartialOrd`, `Ord` | `<`, `>`, `.sort()` | derive |
| `Hash` | usable as a `HashMap` key | derive (along with `Eq`) |
| `Default` | `T::default()` | derive or implement |
| `Display` | `{}` formatting, `.to_string()` | **implement by hand** |
| `From` / `Into` | type conversions; also powers `?` | implement `From`, and you get `Into` for free |
| `Iterator` | `for` loops and all the adapters | chapter 07 |
| `Send` / `Sync` | safe to move or share across threads | automatic (chapter 08) |
| `Serialize` / `Deserialize` | JSON and more (serde crate) | derive (chapter 10) |

**The orphan rule:** you can implement a trait for a type only if **either the trait
or the type is defined in your crate**. So `impl Display for Vec<i32>` isn't
allowed, but `impl Display for MyType` or `impl MyTrait for Vec<i32>` is fine.

## 3. Lifetimes: how long references are valid

Every reference has a lifetime: the region of code where it's valid. Usually the
compiler figures this out (**lifetime elision**). You only write lifetimes when the
compiler can't tell how the input and output references relate:

```rust
fn longest(x: &str, y: &str) -> &str {   // ❌ error[E0106]: missing lifetime specifier
    if x.len() > y.len() { x } else { y }
}
```

Does the returned reference come from `x` or from `y`? The compiler needs to know
so it can check callers. You tell it:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

Read `'a` as: "the result is valid for as long as **both** inputs are valid".
**Lifetime annotations don't change how long anything lives.** They describe
relationships so the compiler can verify them.

```rust
let s1 = String::from("long string");
let result;
{
    let s2 = String::from("xyz");
    result = longest(&s1, &s2);
}                              // s2 dropped here
println!("{result}");          // ❌ s2 does not live long enough
```

### Elision rules (why you rarely write `'a`)

1. Each reference parameter gets its own lifetime.
2. If there's exactly **one** input lifetime, the output gets it.
3. If there's `&self` or `&mut self`, the output gets `self`'s lifetime.

That's why `fn first_word(s: &str) -> &str` needs no annotation.

### Structs holding references

```rust
struct Excerpt<'a> {
    part: &'a str,     // an Excerpt cannot outlive the text it points into
}
```

`'static` means "lives for the entire program", for example string literals.
**Beginner tip:** if lifetimes in structs get painful, store owned data (`String`)
instead. That's usually the right call in services anyway.

## 4. Type this: `examples/06_traits.rs`

```rust
// file: examples/06_traits.rs
use std::fmt;

// ---------- a trait with a default method ----------
trait Describe {
    fn name(&self) -> String;
    fn describe(&self) -> String {
        format!("This is {}", self.name())
    }
}

// ---------- a generic struct ----------
#[derive(Debug, Clone, Copy, PartialEq, Default)]
struct Pair<T> {
    a: T,
    b: T,
}

impl<T: PartialOrd + Copy> Pair<T> {
    fn max(&self) -> T {
        if self.a > self.b { self.a } else { self.b }
    }
}

// ---------- Display by hand ----------
impl<T: fmt::Display> fmt::Display for Pair<T> {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "<{}, {}>", self.a, self.b)
    }
}

// ---------- From conversion ----------
impl<T> From<(T, T)> for Pair<T> {
    fn from((a, b): (T, T)) -> Self {
        Pair { a, b }
    }
}

// ---------- two types implementing one trait ----------
struct Dog;
struct Robot {
    model: u32,
}

impl Describe for Dog {
    fn name(&self) -> String {
        "a dog".into()
    }
}

impl Describe for Robot {
    fn name(&self) -> String {
        format!("robot #{}", self.model)
    }
    fn describe(&self) -> String {
        format!("BEEP. I AM {}", self.name().to_uppercase())
    }
}

// static dispatch: one copy compiled per T
fn announce(item: &impl Describe) {
    println!("[static]  {}", item.describe());
}

// dynamic dispatch: one function, vtable lookup at runtime
fn announce_all(items: &[Box<dyn Describe>]) {
    for i in items {
        println!("[dynamic] {}", i.describe());
    }
}

// generic function with a where clause
fn print_largest<T>(items: &[T])
where
    T: PartialOrd + fmt::Debug,
{
    let mut best = &items[0];
    for it in items {
        if it > best {
            best = it;
        }
    }
    println!("largest of {items:?} is {best:?}");
}

// explicit lifetime: the result borrows from `text`, not from `prefix`
fn after_prefix<'a>(text: &'a str, prefix: &str) -> &'a str {
    text.strip_prefix(prefix).unwrap_or(text)
}

fn main() {
    let p = Pair { a: 3, b: 9 };
    println!("{p} max = {}", p.max());

    let q: Pair<f64> = (2.5, 1.5).into(); // From<(T,T)> gives us .into()
    println!("{q} max = {}", q.max());
    println!("default: {:?}", Pair::<i32>::default());

    announce(&Dog);
    announce(&Robot { model: 42 });
    announce_all(&[Box::new(Dog), Box::new(Robot { model: 7 })]);

    print_largest(&[3, 8, 1]);
    print_largest(&["pear", "apple", "zucchini"]);

    let url = String::from("https://rust-lang.org");
    let host = after_prefix(&url, "https://");
    println!("host = {host}");
}
```

```bash
cargo run --example 06_traits
```

## ✅ Exercises

Create `examples/06_exercises.rs`:

1. Define `trait Area { fn area(&self) -> f64; }`, implement it for `Circle` and
   `Square`, and write `fn total_area(shapes: &[Box<dyn Area>]) -> f64`.
2. Implement `Display` for a `Temperature { celsius: f64 }` struct so that it prints
   `21.5°C`.
3. Write a generic `struct Stack<T> { items: Vec<T> }` with `new`, `push`,
   `pop -> Option<T>`, `peek -> Option<&T>`, `len` and `is_empty`.
4. Write `fn longer<'a>(a: &'a [i32], b: &'a [i32]) -> &'a [i32]`.

<details>
<summary>Solution</summary>

```rust
// file: examples/06_exercises.rs
use std::fmt;

trait Area {
    fn area(&self) -> f64;
}
struct Circle(f64);
struct Square(f64);
impl Area for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.0 * self.0
    }
}
impl Area for Square {
    fn area(&self) -> f64 {
        self.0 * self.0
    }
}
fn total_area(shapes: &[Box<dyn Area>]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

struct Temperature {
    celsius: f64,
}
impl fmt::Display for Temperature {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{:.1}°C", self.celsius)
    }
}

#[derive(Debug, Default)]
struct Stack<T> {
    items: Vec<T>,
}
impl<T> Stack<T> {
    fn new() -> Self {
        Self { items: Vec::new() }
    }
    fn push(&mut self, item: T) {
        self.items.push(item);
    }
    fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }
    fn peek(&self) -> Option<&T> {
        self.items.last()
    }
    fn len(&self) -> usize {
        self.items.len()
    }
    fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

fn longer<'a>(a: &'a [i32], b: &'a [i32]) -> &'a [i32] {
    if a.len() >= b.len() { a } else { b }
}

fn main() {
    let shapes: Vec<Box<dyn Area>> = vec![Box::new(Circle(1.0)), Box::new(Square(2.0))];
    println!("total area = {:.2}", total_area(&shapes));

    println!("{}", Temperature { celsius: 21.5 });

    let mut s = Stack::new();
    s.push("a");
    s.push("b");
    println!("peek={:?} len={} empty={}", s.peek(), s.len(), s.is_empty());
    println!("pop={:?} pop={:?} pop={:?}", s.pop(), s.pop(), s.pop());

    println!("{:?}", longer(&[1, 2], &[3, 4, 5]));
}
```

</details>

Next → [07: Closures and iterators](07-closures-iterators.md)
