# 08: Smart pointers and concurrency

> Official reading: [The Book, ch. 15 "Smart Pointers"](https://doc.rust-lang.org/book/ch15-00-smart-pointers.html),
> [ch. 16 "Fearless Concurrency"](https://doc.rust-lang.org/book/ch16-00-concurrency.html)
> · [`std::sync`](https://doc.rust-lang.org/std/sync/index.html) docs

A web service handles many requests at the same time, and they share state (our
todo list). This chapter covers the exact tools you'll use for that:
**`Arc<RwLock<T>>`**.

## 1. Smart pointers

A smart pointer is a struct that acts like a pointer (it implements `Deref`) and
does extra work, usually in `Drop`. You've already used two: `String` and `Vec`.

### `Box<T>`: put a value on the heap, with a single owner

```rust
let b = Box::new(5);            // the 5 lives on the heap; b owns it
println!("{}", *b + 1);         // `*` dereferences (often automatic)
```

When to use it:

- **Recursive types**, whose size would otherwise be infinite:

  ```rust
  enum List { Cons(i32, Box<List>), Nil }
  ```

- **Trait objects**: `Box<dyn Trait>` (chapter 06).
- Moving a large value without copying it.

### `Rc<T>`: shared ownership, single thread

Ownership rule 2 says "one owner". Sometimes data genuinely has **several owners**,
for example a graph node referenced by several edges. `Rc` (Reference Counted)
counts owners and frees the value when the count reaches zero:

```rust
use std::rc::Rc;

let a = Rc::new(String::from("shared"));
let b = Rc::clone(&a);           // cheap: increments the count, no deep copy
println!("{}", Rc::strong_count(&a));   // 2
```

`Rc<T>` gives **shared, read-only** access. That's the borrowing rules again: many
readers means no writers.

### `RefCell<T>`: interior mutability, checked at runtime

Sometimes you need to mutate through a shared reference. `RefCell` moves the
borrow-checking rules **to runtime**. Breaking them panics instead of failing to
compile:

```rust
use std::cell::RefCell;

let cell = RefCell::new(vec![1]);
cell.borrow_mut().push(2);             // runtime-checked &mut
println!("{:?}", cell.borrow());       // runtime-checked &
```

`Rc<RefCell<T>>` = shared ownership plus mutation, on a **single thread**.

### `Drop`: run code when a value goes out of scope

```rust
struct Guard(&'static str);
impl Drop for Guard {
    fn drop(&mut self) { println!("cleaning up {}", self.0); }
}
```

This pattern is called RAII. Files close, locks release, and connections return to
their pool, all automatically, at the end of a scope.

## 2. Fearless concurrency

### Threads

```rust
use std::thread;

let handle = thread::spawn(|| {
    for i in 1..5 { println!("spawned: {i}"); }
});
handle.join().unwrap();        // wait for it to finish
```

A spawned thread might outlive the function that created it, so its closure must
**own** its data (`move`). Borrowing a local variable from a spawned thread is a
compile error.

### Message passing: channels

> "Do not communicate by sharing memory; instead, share memory by communicating."

```rust
use std::sync::mpsc;           // multiple producer, single consumer

let (tx, rx) = mpsc::channel();
thread::spawn(move || {
    tx.send(String::from("hi")).unwrap();   // ownership of the String moves into the channel
});
println!("{}", rx.recv().unwrap());
```

After `send`, the sending thread can't use the value anymore. The type system
prevents two threads from touching the same data.

### Shared state: `Arc<Mutex<T>>` ⭐

To share **mutable** state between threads you need two things:

1. **`Arc<T>`**: like `Rc`, but the count is **Atomic**, so it's thread-safe.
   Shared ownership across threads.
2. **`Mutex<T>`** or **`RwLock<T>`**: lets only one thread mutate at a time.

```rust
use std::sync::{Arc, Mutex};

let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);           // each thread gets its own Arc handle
    handles.push(thread::spawn(move || {
        let mut num = counter.lock().unwrap();     // lock → MutexGuard (acts like &mut i32)
        *num += 1;
    }));                                           // guard dropped → lock released (RAII!)
}
for h in handles { h.join().unwrap(); }
println!("{}", *counter.lock().unwrap());          // 10
```

In Rust the data lives **inside** the mutex. You can't reach the data without
holding the lock, so "forgot to lock" bugs can't happen.

`RwLock<T>` is the runtime version of the borrowing rules: **many readers
(`.read()`) or one writer (`.write()`)**. It's a good fit when reads are much more
common than writes, as with our todo API.

`.lock().unwrap()`: `lock` returns `Err` only if another thread **panicked** while
holding the lock (the lock is then "poisoned"). Unwrapping is the common choice.

### `Send` and `Sync`: how the compiler knows

| Marker trait | Meaning | Not implemented by |
|---|---|---|
| `Send` | Safe to **move** to another thread | `Rc<T>` (its counter isn't atomic) |
| `Sync` | Safe to **share** (`&T`) between threads | `RefCell<T>`, `Cell<T>` |

They're implemented **automatically** based on a type's fields. If you try to send
an `Rc` to a thread:

```
error[E0277]: `Rc<i32>` cannot be sent between threads safely
```

That's a data race caught at compile time. This is what "fearless concurrency"
means.

### Cheat sheet

| Need | Single-threaded | Multi-threaded |
|---|---|---|
| Single owner on the heap | `Box<T>` | `Box<T>` |
| Shared ownership, read-only | `Rc<T>` | `Arc<T>` |
| Shared ownership plus mutation | `Rc<RefCell<T>>` | `Arc<Mutex<T>>` / `Arc<RwLock<T>>` |
| Simple counters and flags | `Cell<T>` | `AtomicUsize`, `AtomicBool` |

## 3. Type this: `examples/08_concurrency.rs`

```rust
// file: examples/08_concurrency.rs
use std::collections::HashMap;
use std::rc::Rc;
use std::sync::{Arc, Mutex, RwLock, mpsc};
use std::thread;
use std::time::Duration;

struct Noisy(&'static str);
impl Drop for Noisy {
    fn drop(&mut self) {
        println!("  drop({})", self.0);
    }
}

fn main() {
    // --- Box + Drop ---
    println!("== Box & Drop");
    {
        let _a = Box::new(Noisy("boxed"));
        let _b = Noisy("stack");
        println!("  leaving scope...");
    } // dropped in reverse order of declaration

    // --- Rc ---
    println!("== Rc");
    let shared = Rc::new(String::from("config"));
    let r2 = Rc::clone(&shared);
    println!("  count = {}, value = {r2}", Rc::strong_count(&shared));

    // --- threads + channel ---
    println!("== channels");
    let (tx, rx) = mpsc::channel();
    for id in 0..3 {
        let tx = tx.clone();
        thread::spawn(move || {
            thread::sleep(Duration::from_millis(30 * (3 - id)));
            tx.send(format!("worker {id} done")).unwrap();
        });
    }
    drop(tx); // close our copy so the receiving loop ends when the workers finish
    for msg in rx {
        println!("  {msg}");
    }

    // --- Arc<Mutex<T>> ---
    println!("== Arc<Mutex>");
    let counter = Arc::new(Mutex::new(0u32));
    let handles: Vec<_> = (0..8)
        .map(|_| {
            let counter = Arc::clone(&counter);
            thread::spawn(move || {
                for _ in 0..1000 {
                    *counter.lock().unwrap() += 1;
                }
            })
        })
        .collect();
    for h in handles {
        h.join().unwrap();
    }
    println!("  counter = {}", *counter.lock().unwrap()); // always 8000

    // --- Arc<RwLock<HashMap>>: a tiny in-memory "database" ---
    println!("== Arc<RwLock> (preview of the service's state)");
    let db: Arc<RwLock<HashMap<u32, String>>> = Arc::new(RwLock::new(HashMap::new()));

    let writers: Vec<_> = (1..=3)
        .map(|id| {
            let db = Arc::clone(&db);
            thread::spawn(move || {
                db.write().unwrap().insert(id, format!("user-{id}"));
            })
        })
        .collect();
    for w in writers {
        w.join().unwrap();
    }

    let readers: Vec<_> = (1..=3)
        .map(|id| {
            let db = Arc::clone(&db);
            thread::spawn(move || {
                let guard = db.read().unwrap(); // many readers at once are allowed
                format!("reader sees {id} -> {:?}", guard.get(&id))
            })
        })
        .collect();
    for r in readers {
        println!("  {}", r.join().unwrap());
    }

    // Uncomment to see a compile-time data-race check:
    // let rc = Rc::new(5);
    // thread::spawn(move || println!("{rc}"));   // ❌ E0277: Rc cannot be sent between threads
}
```

```bash
cargo run --example 08_concurrency
```

## ✅ Exercises

Create `examples/08_exercises.rs`:

1. Spawn 4 threads that each compute the sum of one quarter of `1..=1_000_000` and
   send their partial sum through a channel. The main thread adds the partial sums.
2. Do the same using a shared `Arc<Mutex<u64>>` instead of a channel.
3. Bonus: try `std::thread::scope`. It lets threads **borrow** local data without
   `Arc`, because every scoped thread is guaranteed to finish before the scope ends.

<details>
<summary>Solution</summary>

```rust
// file: examples/08_exercises.rs
use std::sync::{Arc, Mutex, mpsc};
use std::thread;

const N: u64 = 1_000_000;
const PARTS: u64 = 4;

fn range_for(i: u64) -> std::ops::RangeInclusive<u64> {
    let size = N / PARTS;
    (i * size + 1)..=((i + 1) * size)
}

fn main() {
    // 1. channels
    let (tx, rx) = mpsc::channel();
    for i in 0..PARTS {
        let tx = tx.clone();
        thread::spawn(move || tx.send(range_for(i).sum::<u64>()).unwrap());
    }
    drop(tx);
    let total: u64 = rx.iter().sum();
    println!("channel total = {total}");

    // 2. shared mutex
    let acc = Arc::new(Mutex::new(0u64));
    let hs: Vec<_> = (0..PARTS)
        .map(|i| {
            let acc = Arc::clone(&acc);
            thread::spawn(move || {
                let part: u64 = range_for(i).sum();
                *acc.lock().unwrap() += part;
            })
        })
        .collect();
    hs.into_iter().for_each(|h| h.join().unwrap());
    println!("mutex total   = {}", *acc.lock().unwrap());

    // 3. scoped threads borrow `data` directly
    let data: Vec<u64> = (1..=N).collect();
    let chunk = data.len() / PARTS as usize;
    let total: u64 = thread::scope(|s| {
        let hs: Vec<_> = data
            .chunks(chunk)
            .map(|c| s.spawn(move || c.iter().sum::<u64>()))
            .collect();
        hs.into_iter().map(|h| h.join().unwrap()).sum()
    });
    println!("scoped total  = {total}");
}
```

</details>

Next → [09: Async Rust and Tokio](09-async-tokio.md)
