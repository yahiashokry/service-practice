# 03: Structs, enums and pattern matching

> Official reading: [The Book, ch. 5 "Structs"](https://doc.rust-lang.org/book/ch05-00-structs.html),
> [ch. 6 "Enums and Pattern Matching"](https://doc.rust-lang.org/book/ch06-00-enums.html),
> [ch. 19 "Patterns and Matching"](https://doc.rust-lang.org/book/ch19-00-patterns.html)

Structs and enums are how you **model your domain** in Rust. Together with `match`
they're the main tool for "making invalid states unrepresentable".

## 1. Structs

```rust
#[derive(Debug, Clone, PartialEq)]   // auto-implement common traits (see ch. 06)
struct User {
    username: String,                 // owned data in fields (see ch. 02)
    email: String,
    active: bool,
    sign_in_count: u64,
}

let mut user = User {
    username: String::from("yahia"),
    email: String::from("y@example.com"),
    active: true,
    sign_in_count: 1,
};
user.sign_in_count += 1;              // whole instance must be `mut`

// field init shorthand + struct update syntax
fn new_user(username: String, email: String) -> User {
    User { username, email, active: true, sign_in_count: 0 }
}
let user2 = User { email: String::from("other@example.com"), ..user.clone() };
```

Other struct forms:

```rust
struct Point(i32, i32);   // tuple struct: fields are .0, .1
struct Meters(f64);       // "newtype": a distinct type wrapping f64
struct Marker;            // unit struct: no data
```

**Newtypes** are a powerful idiom: `Meters` and `Seconds` can both wrap `f64`, but
the compiler won't let you mix them up.

### Methods and associated functions: `impl`

```rust
impl User {
    // associated function (no self): called as User::new(...)
    // `new` is a convention, not a keyword
    fn new(username: &str, email: &str) -> Self {
        Self {
            username: username.to_string(),
            email: email.to_string(),
            active: true,
            sign_in_count: 0,
        }
    }

    fn greeting(&self) -> String {          // &self = borrow immutably
        format!("Hi, {}!", self.username)
    }

    fn record_login(&mut self) {            // &mut self = borrow mutably
        self.sign_in_count += 1;
    }

    fn into_email(self) -> String {         // self = take ownership (consumes the User)
        self.email
    }
}
```

The `self` forms are the same ownership choices you learned in chapter 02, applied
to methods.

## 2. Enums: a type that is one of several variants

Rust enums are much more powerful than C or Java enums: **each variant can carry
different data**.

```rust
enum Message {
    Quit,                          // no data
    Move { x: i32, y: i32 },       // named fields
    Write(String),                 // one value
    ChangeColor(u8, u8, u8),       // several values
}
```

### `Option<T>`: Rust has no null

```rust
enum Option<T> {    // from the standard library, always in scope
    Some(T),
    None,
}
```

Instead of "a value that might secretly be null", Rust makes possible absence
**part of the type**. An `Option<String>` can't be used as a `String`. The compiler
forces you to handle `None`:

```rust
let maybe: Option<i32> = Some(5);
let x: i32 = maybe + 1;              // ❌ can't add Option<i32> and i32
let x: i32 = maybe.unwrap_or(0) + 1; // ✅ you decided what None means
```

This removes the "billion-dollar mistake" (null pointer errors) from the language.

## 3. `match`: exhaustive pattern matching

```rust
fn describe(msg: &Message) -> String {
    match msg {
        Message::Quit => "quit".to_string(),
        Message::Move { x, y } => format!("move to ({x}, {y})"),
        Message::Write(text) => format!("write '{text}'"),
        Message::ChangeColor(r, g, b) => format!("color #{r:02x}{g:02x}{b:02x}"),
    }
}
```

`match` is **exhaustive**. If you forget a variant, the code doesn't compile. If you
later add a variant to the enum, the compiler points you to **every `match`** you
need to update. This is a big part of why refactoring Rust is safe.

Pattern features:

```rust
match n {
    0 => "zero",
    1 | 2 => "one or two",           // or-patterns
    3..=9 => "single digit",          // ranges
    x if x < 0 => "negative",         // match guard
    _ => "big",                       // wildcard: everything else
}
```

### `if let` and `let else`: when you care about one case

```rust
let config_port: Option<u16> = Some(8080);

if let Some(port) = config_port {
    println!("port {port}");
}

// let-else: destructure or bail out early. Very common in real code.
fn parse_port(s: &str) -> u16 {
    let Ok(port) = s.parse::<u16>() else {
        return 3000;                  // the else block must diverge (return/break/panic)
    };
    port
}
```

### Destructuring works everywhere

```rust
let User { username, active, .. } = user;   // structs (.. ignores the other fields)
let (a, b) = (1, 2);                         // tuples
for (key, value) in map { }                  // in for loops
fn print_point(&(x, y): &(i32, i32)) { }     // in function parameters
```

## 4. Type this: `examples/03_shapes.rs`

```rust
// file: examples/03_shapes.rs
use std::f64::consts::PI;

#[derive(Debug, Clone, PartialEq)]
enum Shape {
    Circle { radius: f64 },
    Rectangle { width: f64, height: f64 },
    Triangle(f64, f64, f64),
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { radius } => PI * radius * radius,
            Shape::Rectangle { width, height } => width * height,
            Shape::Triangle(a, b, c) => {
                // Heron's formula
                let s = (a + b + c) / 2.0;
                (s * (s - a) * (s - b) * (s - c)).sqrt()
            }
        }
    }

    fn name(&self) -> &'static str {
        match self {
            Shape::Circle { .. } => "circle",
            Shape::Rectangle { width, height } if width == height => "square",
            Shape::Rectangle { .. } => "rectangle",
            Shape::Triangle(..) => "triangle",
        }
    }
}

#[derive(Debug)]
struct Canvas {
    name: String,
    shapes: Vec<Shape>,
}

impl Canvas {
    fn new(name: &str) -> Self {
        Self {
            name: name.to_string(),
            shapes: Vec::new(),
        }
    }

    fn add(&mut self, shape: Shape) {
        self.shapes.push(shape);
    }

    fn largest(&self) -> Option<&Shape> {
        let mut best: Option<&Shape> = None;
        for s in &self.shapes {
            match best {
                None => best = Some(s),
                Some(b) if s.area() > b.area() => best = Some(s),
                Some(_) => {}
            }
        }
        best
    }
}

fn main() {
    let mut canvas = Canvas::new("demo");
    canvas.add(Shape::Circle { radius: 1.5 });
    canvas.add(Shape::Rectangle { width: 2.0, height: 2.0 });
    canvas.add(Shape::Rectangle { width: 3.0, height: 1.0 });
    canvas.add(Shape::Triangle(3.0, 4.0, 5.0));

    println!("canvas '{}':", canvas.name);
    for s in &canvas.shapes {
        println!("  {:<10} area = {:>6.2}", s.name(), s.area());
    }

    if let Some(big) = canvas.largest() {
        println!("largest: {big:?}");
    }

    let empty = Canvas::new("empty");
    let Some(_) = empty.largest() else {
        println!("'{}' has no shapes", empty.name);
        return;
    };
}
```

```bash
cargo run --example 03_shapes
```

Experiment: add a `Square(f64)` variant to `Shape` and run `cargo check`. The
compiler lists every `match` you need to update.

## ✅ Exercises

Create `examples/03_exercises.rs`:

1. Model a traffic light: `enum Light { Red, Yellow, Green }` with
   `fn next(&self) -> Light` and `fn duration_secs(&self) -> u32`.
2. Model an HTTP-like response:
   `enum Response { Ok(String), NotFound, Error { code: u16, reason: String } }`, plus
   `fn status_code(&self) -> u16`.
3. Write `fn safe_divide(a: f64, b: f64) -> Option<f64>` and handle it with `match`,
   then with `if let`, then with `unwrap_or`.

<details>
<summary>Solution</summary>

```rust
// file: examples/03_exercises.rs
#[derive(Debug, Clone, Copy, PartialEq)]
enum Light {
    Red,
    Yellow,
    Green,
}

impl Light {
    fn next(&self) -> Light {
        match self {
            Light::Red => Light::Green,
            Light::Green => Light::Yellow,
            Light::Yellow => Light::Red,
        }
    }
    fn duration_secs(&self) -> u32 {
        match self {
            Light::Red => 30,
            Light::Yellow => 5,
            Light::Green => 25,
        }
    }
}

#[derive(Debug)]
enum Response {
    Ok(String),
    NotFound,
    Error { code: u16, reason: String },
}

impl Response {
    fn status_code(&self) -> u16 {
        match self {
            Response::Ok(_) => 200,
            Response::NotFound => 404,
            Response::Error { code, .. } => *code,
        }
    }
}

fn safe_divide(a: f64, b: f64) -> Option<f64> {
    if b == 0.0 { None } else { Some(a / b) }
}

fn main() {
    let mut light = Light::Red;
    for _ in 0..4 {
        println!("{light:?} for {}s", light.duration_secs());
        light = light.next();
    }

    let responses = [
        Response::Ok("hello".into()),
        Response::NotFound,
        Response::Error { code: 503, reason: "maintenance".into() },
    ];
    for r in &responses {
        println!("{} <- {r:?}", r.status_code());
        match r {
            Response::Ok(body) => println!("    body: {body}"),
            Response::Error { reason, .. } => println!("    reason: {reason}"),
            Response::NotFound => {}
        }
    }

    match safe_divide(10.0, 4.0) {
        Some(v) => println!("10/4 = {v}"),
        None => println!("division by zero"),
    }
    if let Some(v) = safe_divide(1.0, 0.0) {
        println!("never printed: {v}");
    }
    println!("fallback: {}", safe_divide(1.0, 0.0).unwrap_or(f64::NAN));
}
```

</details>

Next → [04: Modules, crates and tests](04-modules-crates-tests.md)
