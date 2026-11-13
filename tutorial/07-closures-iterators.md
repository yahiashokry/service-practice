# 07: Closures and iterators

> Official reading: [The Book, ch. 13 "Functional Language Features"](https://doc.rust-lang.org/book/ch13-00-functional-features.html)
> · std docs: [`Iterator` trait](https://doc.rust-lang.org/std/iter/trait.Iterator.html) (skim the list of methods; you'll come back to it often)

## 1. Closures: anonymous functions that capture their environment

```rust
let add_one = |x: i32| x + 1;          // parameter types often inferred
let add_one = |x| x + 1;
let greet = || println!("hi");
let multi = |a, b| {
    let s = a + b;
    s * 2
};

let offset = 10;
let shift = |x| x + offset;            // captures `offset` from the surrounding scope
println!("{}", shift(5));              // 15
```

### How closures capture: ownership again

A closure captures each variable in the **least powerful way** that works:

| The closure body... | Captures by | Implements trait |
|---|---|---|
| only reads the variable | `&T` | `Fn` (callable many times) |
| mutates the variable | `&mut T` | `FnMut` (callable many times, needs `mut`) |
| moves the variable out (consumes it) | `T` | `FnOnce` (callable once) |

```rust
let mut count = 0;
let mut inc = || count += 1;   // FnMut: borrows `count` mutably
inc();
inc();
println!("{count}");           // 2 (the mutable borrow ended after the last inc())
```

### `move` closures

`move` forces the closure to **take ownership** of everything it captures. You need
it whenever the closure outlives the current scope, for example when it's sent to
another thread or spawned as an async task (chapters 08 and 09):

```rust
let name = String::from("worker");
let handle = std::thread::spawn(move || {
    println!("hello from {name}");   // `name` is owned by the closure now
});
handle.join().unwrap();
```

### Closures as parameters

```rust
fn apply<F: Fn(i32) -> i32>(f: F, x: i32) -> i32 { f(x) }

fn apply_twice(f: impl Fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

fn make_multiplier(n: i32) -> impl Fn(i32) -> i32 {
    move |x| x * n                   // returns a closure; must `move` n in
}
```

## 2. Iterators

An iterator is anything implementing the `Iterator` trait:

```rust
trait Iterator {
    type Item;                                   // associated type
    fn next(&mut self) -> Option<Self::Item>;    // the only required method
    // ...plus 70+ provided methods built on top of next()
}
```

Three ways to get one from a collection, matching the ownership forms:

```rust
v.iter()        // yields &T          (borrow)
v.iter_mut()    // yields &mut T      (mutable borrow)
v.into_iter()   // yields T           (takes ownership of v)
```

### Iterators are lazy

**Adapters** (`map`, `filter`, ...) build a new iterator and **do nothing** until a
**consumer** (`collect`, `sum`, `for`, ...) pulls values through:

```rust
let v = vec![1, 2, 3];
v.iter().map(|x| x * 2);                       // ⚠ warning: unused iterator, nothing runs
let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();   // ✅ runs now
```

### The adapters and consumers you'll use every day

```rust
let nums = vec![1, 2, 3, 4, 5, 6, 7, 8, 9, 10];

// adapters (lazy)
nums.iter().map(|x| x * x)
nums.iter().filter(|&&x| x % 2 == 0)      // filter gets &&i32 here; patterns peel the refs
nums.iter().filter_map(|x| some_option(x))
nums.iter().enumerate()                   // (index, item)
nums.iter().zip(other.iter())             // pairs
nums.iter().skip(2).take(3)
nums.iter().rev()
nums.iter().chain(more.iter())
nums.iter().flat_map(|x| vec![*x, *x])
nums.windows(2)  /  nums.chunks(3)        // slice methods that return iterators

// consumers (drive the iteration)
.collect::<Vec<_>>()   .collect::<HashMap<_, _>>()   .collect::<String>()
.sum::<i32>()   .count()   .min()   .max()   .max_by_key(|x| ..)
.any(|x| ..)    .all(|x| ..)   .find(|x| ..)   .position(|x| ..)
.fold(0, |acc, x| acc + x)
.for_each(|x| ..)
```

`collect` is driven by the **type you ask for**. It can even collect an iterator of
`Result`s into a single `Result`, stopping at the first error:

```rust
let parsed: Result<Vec<i32>, _> = ["1", "2", "x"].iter().map(|s| s.parse::<i32>()).collect();
// Err(ParseIntError)  — great for validating input
```

### Performance: zero-cost abstractions

Iterator chains compile down to the same machine code as hand-written loops, and
sometimes faster code, because bounds checks can be removed. Idiomatic Rust
prefers iterator chains over index loops.

## 3. Type this: `examples/07_iterators.rs`

```rust
// file: examples/07_iterators.rs
use std::collections::HashMap;

#[derive(Debug, Clone)]
struct Order {
    customer: String,
    item: String,
    qty: u32,
    price_cents: u32,
}

impl Order {
    fn new(customer: &str, item: &str, qty: u32, price_cents: u32) -> Self {
        Self {
            customer: customer.into(),
            item: item.into(),
            qty,
            price_cents,
        }
    }
    fn total(&self) -> u32 {
        self.qty * self.price_cents
    }
}

// A custom iterator: implementing `next` gives us every adapter for free
struct Countdown(u32);

impl Iterator for Countdown {
    type Item = u32;
    fn next(&mut self) -> Option<u32> {
        if self.0 == 0 {
            None
        } else {
            self.0 -= 1;
            Some(self.0 + 1)
        }
    }
}

fn make_discount(percent: u32) -> impl Fn(u32) -> u32 {
    move |cents| cents * (100 - percent) / 100
}

fn main() {
    let orders = [
        Order::new("ana", "book", 2, 1500),
        Order::new("bob", "pen", 10, 120),
        Order::new("ana", "lamp", 1, 4500),
        Order::new("cy", "book", 1, 1500),
        Order::new("bob", "desk", 1, 25000),
    ];

    // map + sum
    let revenue: u32 = orders.iter().map(Order::total).sum();
    println!("revenue: ${:.2}", revenue as f64 / 100.0);

    // filter + map + collect
    let big: Vec<&str> = orders
        .iter()
        .filter(|o| o.total() >= 4000)
        .map(|o| o.item.as_str())
        .collect();
    println!("big-ticket items: {big:?}");

    // fold into a HashMap: spend per customer
    let per_customer: HashMap<&str, u32> = orders.iter().fold(HashMap::new(), |mut acc, o| {
        *acc.entry(o.customer.as_str()).or_insert(0) += o.total();
        acc
    });
    let mut ranking: Vec<_> = per_customer.into_iter().collect();
    ranking.sort_by_key(|&(_, total)| std::cmp::Reverse(total)); // highest first
    println!("ranking: {ranking:?}");

    // max_by_key, any, all, find, position
    let priciest = orders.iter().max_by_key(|o| o.price_cents).unwrap();
    println!("priciest unit: {} ({})", priciest.item, priciest.price_cents);
    println!("anyone bought a pen? {}", orders.iter().any(|o| o.item == "pen"));
    println!("all qty > 0? {}", orders.iter().all(|o| o.qty > 0));
    println!("first of cy: {:?}", orders.iter().find(|o| o.customer == "cy").map(|o| &o.item));

    // closures capturing environment
    let discount = make_discount(20);
    let discounted: Vec<u32> = orders.iter().map(|o| discount(o.total())).collect();
    println!("discounted totals: {discounted:?}");

    // FnMut: a closure that mutates captured state
    let mut seen = 0;
    orders.iter().for_each(|_| seen += 1);
    println!("seen {seen} orders");

    // custom iterator + adapters
    let launch: Vec<String> = Countdown(5).map(|n| n.to_string()).collect();
    println!("{} liftoff!", launch.join("... "));

    // collect into Result: all-or-nothing parsing
    let good: Result<Vec<u32>, _> = "1,2,3".split(',').map(str::parse::<u32>).collect();
    let bad: Result<Vec<u32>, _> = "1,x,3".split(',').map(str::parse::<u32>).collect();
    println!("good = {good:?}\nbad  = {bad:?}");

    // zip + windows
    let days = ["mon", "tue", "wed", "thu"];
    let temps = [18, 21, 17, 23];
    for (d, t) in days.iter().zip(temps.iter()) {
        print!("{d}:{t}° ");
    }
    println!();
    let deltas: Vec<i32> = temps.windows(2).map(|w| w[1] - w[0]).collect();
    println!("day-to-day change: {deltas:?}");
}
```

```bash
cargo run --example 07_iterators
```

## ✅ Exercises

Create `examples/07_exercises.rs`. Solve each **without** writing a `for` loop:

1. Sum the squares of all odd numbers in `1..=100`.
2. Given `Vec<String>` names, produce a single `String` of their uppercase
   initials, e.g. `["ada", "linus", "grace"]` → `"ALG"`.
3. Given a sentence, return the longest word (use `max_by_key`).
4. Implement a `Fibonacci` struct that implements `Iterator`, and print the first
   10 even Fibonacci numbers using `.filter()` and `.take()`.
5. Write `fn compose(f: impl Fn(i32) -> i32, g: impl Fn(i32) -> i32) -> impl Fn(i32) -> i32`.

<details>
<summary>Solution</summary>

```rust
// file: examples/07_exercises.rs
struct Fibonacci {
    a: u64,
    b: u64,
}

impl Iterator for Fibonacci {
    type Item = u64;
    fn next(&mut self) -> Option<u64> {
        let out = self.a;
        (self.a, self.b) = (self.b, self.a + self.b);
        Some(out) // infinite iterator: always Some
    }
}

fn compose(f: impl Fn(i32) -> i32, g: impl Fn(i32) -> i32) -> impl Fn(i32) -> i32 {
    move |x| g(f(x))
}

fn main() {
    let s: u32 = (1..=100).filter(|n| n % 2 == 1).map(|n| n * n).sum();
    println!("1) {s}");

    let names = ["ada".to_string(), "linus".into(), "grace".into()];
    let initials: String = names
        .iter()
        .filter_map(|n| n.chars().next())
        .map(|c| c.to_ascii_uppercase())
        .collect();
    println!("2) {initials}");

    let sentence = "iterators make rust code expressive";
    let longest = sentence.split_whitespace().max_by_key(|w| w.len());
    println!("3) {longest:?}");

    let evens: Vec<u64> = Fibonacci { a: 0, b: 1 }
        .filter(|n| n % 2 == 0)
        .take(10)
        .collect();
    println!("4) {evens:?}");

    let add_then_double = compose(|x| x + 1, |x| x * 2);
    println!("5) {}", add_then_double(5)); // (5+1)*2 = 12
}
```

</details>

Next → [08: Smart pointers and concurrency](08-smart-pointers-concurrency.md)
