# 09: Async Rust and Tokio

> Official reading: [The Book, ch. 17 "Async and Await"](https://doc.rust-lang.org/book/ch17-00-async-await.html),
> [Asynchronous Programming in Rust (async book)](https://rust-lang.github.io/async-book/),
> [Tokio tutorial](https://tokio.rs/tokio/tutorial) (read at least "Hello Tokio", "Spawning", "Shared state", "Channels")

## 1. Why async?

A web service spends most of its time **waiting**: for the network, the database,
or the disk. With one OS thread per connection, 10,000 connections means 10,000
threads, each with its own stack and context-switching cost.

**Async** lets a few threads juggle thousands of tasks. When a task would wait, it
**yields**, and the thread runs another task in the meantime.

| | Threads (ch. 08) | Async tasks |
|---|---|---|
| Good for | CPU-heavy work | I/O-heavy work (servers, clients) |
| Cost per unit | OS thread (~MBs of stack) | Task (~bytes to KBs) |
| Scheduling | OS, preemptive | Runtime, cooperative (at each `.await`) |

## 2. `async`, `.await` and futures

```rust
async fn fetch_user(id: u32) -> String {
    // ... imagine a network call here ...
    format!("user {id}")
}
```

An `async fn` **doesn't run when you call it**. It returns a **`Future`**: a value
that represents work which will finish later.

```rust
let fut = fetch_user(1);       // nothing has happened yet: futures are LAZY
let user = fut.await;          // .await drives it to completion, yielding while it waits
```

- `.await` can only be used **inside** an `async` function or block.
- A future does nothing unless something **polls** it. That something is an
  **async runtime**.

### The runtime: Tokio

Rust's standard library defines `Future` but **ships no runtime**. You choose one.
**Tokio** is the de-facto standard, and axum (chapter 10) is built on it.

```bash
cargo add tokio --features full
```

```rust
#[tokio::main]                 // macro: builds a multi-threaded runtime and runs main on it
async fn main() {
    let user = fetch_user(1).await;
    println!("{user}");
}
```

## 3. Concurrency with async

```rust
use tokio::time::{sleep, Duration};

async fn work(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;      // async sleep: yields the thread
    format!("{name} done")
}

// Sequential: ~300ms
let a = work("a", 100).await;
let b = work("b", 200).await;

// Concurrent within one task: ~200ms
let (a, b) = tokio::join!(work("a", 100), work("b", 200));

// Spawned tasks: run independently, possibly on other threads
let handle = tokio::spawn(async move {           // `move` again: the task must own its data
    work("bg", 50).await
});
let result = handle.await.unwrap();              // JoinHandle → Result<T, JoinError>

// Race: take whichever finishes first. This is how timeouts work.
tokio::select! {
    r = work("slow", 1000) => println!("{r}"),
    _ = sleep(Duration::from_millis(100)) => println!("timed out"),
}
// or simply: tokio::time::timeout(Duration::from_millis(100), fut).await
```

`tokio::spawn` requires the future to be **`'static`** (it owns all its data, with no
borrows of local variables) and **`Send`** (it may move between threads). That's
why you'll see `Arc` and `move` everywhere in async code. It's the same reasoning
as `thread::spawn` in chapter 08.

## 4. The golden rule: never block the runtime ⚠️

Tokio runs many tasks on a few threads. If one task **blocks** a thread, every task
waiting on that thread stalls.

```rust
std::thread::sleep(Duration::from_secs(1));      // ❌ blocks the worker thread
tokio::time::sleep(Duration::from_secs(1)).await; // ✅ yields

std::fs::read_to_string("f")                      // ⚠ blocking I/O
tokio::fs::read_to_string("f").await              // ✅ async I/O

// heavy CPU work or a blocking library? Move it off the async threads:
let v = tokio::task::spawn_blocking(|| expensive_computation()).await.unwrap();
```

### `std::sync::Mutex` or `tokio::sync::Mutex`?

- Use **`std::sync::Mutex` / `RwLock`** (yes, in async code) when you hold the lock
  briefly and **never across an `.await`**. This is the common case, and the one
  we use in the service.
- Use **`tokio::sync::Mutex`** only when you must hold the lock across an `.await`.

The compiler helps here: a `std::sync::MutexGuard` is not `Send`, so holding one
across an `.await` inside `tokio::spawn` fails to compile.

## 5. Type this: `examples/09_async.rs`

First, add the dependency. This edits `Cargo.toml`:

```bash
cargo add tokio --features full
```

This example starts a **real TCP echo server** in a background task, connects to it
as a client, and shows the async patterns above.

```rust
// file: examples/09_async.rs
use std::sync::{Arc, Mutex};
use std::time::Instant;

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::{Duration, sleep, timeout};

async fn work(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;
    format!("{name} finished after {ms}ms")
}

/// Echo server: uppercases every line it receives.
async fn run_echo_server(listener: TcpListener, served: Arc<Mutex<u32>>) {
    loop {
        let (socket, addr) = listener.accept().await.unwrap();
        let served = Arc::clone(&served);

        // one task per connection: thousands of these are cheap
        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut lines = BufReader::new(reader).lines();

            while let Ok(Some(line)) = lines.next_line().await {
                let reply = format!("{}\n", line.to_uppercase());
                if writer.write_all(reply.as_bytes()).await.is_err() {
                    break;
                }
                // lock held briefly, never across an .await, so std Mutex is fine
                *served.lock().unwrap() += 1;
            }
            println!("  [server] {addr} disconnected");
        });
    }
}

#[tokio::main]
async fn main() {
    // --- sequential vs concurrent ---
    let t = Instant::now();
    work("a", 100).await;
    work("b", 100).await;
    println!("sequential: {:?}", t.elapsed());

    let t = Instant::now();
    let (a, b) = tokio::join!(work("a", 100), work("b", 100));
    println!("join!:      {:?}  ({a} / {b})", t.elapsed());

    // --- spawning many tasks ---
    let t = Instant::now();
    let handles: Vec<_> = (0..1000)
        .map(|i| tokio::spawn(async move { work("task", 50).await.len() + i }))
        .collect();
    let mut sum = 0;
    for h in handles {
        sum += h.await.unwrap();
    }
    println!("1000 tasks: {:?} (checksum {sum})", t.elapsed());

    // --- timeouts ---
    match timeout(Duration::from_millis(50), work("slow", 500)).await {
        Ok(v) => println!("{v}"),
        Err(_) => println!("timeout: slow work cancelled"),
    }

    // --- a real TCP server + client ---
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap(); // port 0 = pick any free port
    let addr = listener.local_addr().unwrap();
    let served = Arc::new(Mutex::new(0));
    tokio::spawn(run_echo_server(listener, Arc::clone(&served)));
    println!("echo server on {addr}");

    let stream = TcpStream::connect(addr).await.unwrap();
    let (reader, mut writer) = stream.into_split();
    let mut replies = BufReader::new(reader).lines();

    for msg in ["hello", "async rust", "tokio"] {
        writer.write_all(format!("{msg}\n").as_bytes()).await.unwrap();
        let reply = replies.next_line().await.unwrap().unwrap();
        println!("  [client] sent {msg:?}, got {reply:?}");
    }
    drop(writer); // close our side, so the server sees EOF
    sleep(Duration::from_millis(50)).await;

    println!("server handled {} lines", *served.lock().unwrap());
}
```

```bash
cargo run --example 09_async
```

Look at the timings. 1000 tasks that each sleep 50ms finish in roughly 50ms total,
not 50 seconds.

## 6. What a web framework adds on top

The echo server above *is* a network service. A real HTTP service also needs to
parse HTTP, route `GET /todos/3` to the right function, turn JSON into structs and
back, return proper status codes, and add logging and middleware.

**axum** does all of that, built on **tokio** (runtime), **hyper** (HTTP), and
**tower** (middleware). That's the next chapter.

> Curious how it works under the hood? The Book's final project,
> [ch. 21 "Building a Multithreaded Web Server"](https://doc.rust-lang.org/book/ch21-00-final-project-a-web-server.html),
> builds a tiny HTTP server from scratch with only `std`. It's a great follow-up.

## ✅ Exercises

Create `examples/09_exercises.rs`:

1. Write `async fn fetch(id: u32) -> u32` that sleeps for `id * 10` ms and returns
   `id * id`. Spawn tasks for ids 1..=20 and collect the results in order.
2. Use `tokio::select!` in a loop to print "tick" every 100ms until 550ms have passed.
   Hint: `tokio::time::interval` and a `sleep` future pinned with `tokio::pin!`.
3. Use a `tokio::sync::mpsc` channel: 3 producer tasks each send 5 numbers, and one
   consumer task prints them and returns the total.

<details>
<summary>Solution</summary>

```rust
// file: examples/09_exercises.rs
use tokio::sync::mpsc;
use tokio::time::{Duration, interval, sleep};

async fn fetch(id: u32) -> u32 {
    sleep(Duration::from_millis(id as u64 * 10)).await;
    id * id
}

#[tokio::main]
async fn main() {
    // 1.
    let handles: Vec<_> = (1..=20).map(|id| tokio::spawn(fetch(id))).collect();
    let mut results = Vec::new();
    for h in handles {
        results.push(h.await.unwrap());
    }
    println!("1) {results:?}");

    // 2.
    let mut ticker = interval(Duration::from_millis(100));
    let deadline = sleep(Duration::from_millis(550));
    tokio::pin!(deadline);
    loop {
        tokio::select! {
            _ = ticker.tick() => println!("2) tick"),
            _ = &mut deadline => { println!("2) done"); break; }
        }
    }

    // 3.
    let (tx, mut rx) = mpsc::channel::<u32>(16);
    for p in 0..3 {
        let tx = tx.clone();
        tokio::spawn(async move {
            for i in 0..5 {
                tx.send(p * 100 + i).await.unwrap();
            }
        });
    }
    drop(tx);
    let consumer = tokio::spawn(async move {
        let mut total = 0;
        while let Some(n) = rx.recv().await {
            total += n;
        }
        total
    });
    println!("3) total = {}", consumer.await.unwrap());
}
```

</details>

Next → [10: Build the service](10-build-the-service.md) 🚀
