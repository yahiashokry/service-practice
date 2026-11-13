# 10: Build the service 🚀

> Official docs used in this chapter:
> [axum](https://docs.rs/axum/latest/axum/) (read the crate's front page) ·
> [axum examples](https://github.com/tokio-rs/axum/tree/main/examples) ·
> [Tokio](https://tokio.rs/tokio/tutorial) · [Serde](https://serde.rs/) ·
> [tracing](https://docs.rs/tracing/latest/tracing/) · [thiserror](https://docs.rs/thiserror/latest/thiserror/) ·
> [tower-http](https://docs.rs/tower-http/latest/tower_http/)

You'll build a **Todo REST API** in `src/`, step by step. Every step ends with
something you can run.

| Method | Path | Does |
|--------|------|------|
| `GET` | `/health` | liveness check |
| `GET` | `/todos` | list all todos |
| `POST` | `/todos` | create a todo `{"title": "..."}` |
| `GET` | `/todos/{id}` | get one |
| `PATCH` | `/todos/{id}` | update `{"title"?: "...", "done"?: true}` |
| `DELETE` | `/todos/{id}` | delete |

Final layout:

```
src/
├── main.rs     ← startup: logging, config, server, graceful shutdown
├── models.rs   ← data types (serde)
├── state.rs    ← shared in-memory "database": Arc<RwLock<...>>
├── error.rs    ← AppError → HTTP response
└── routes.rs   ← router + handlers + tests
```

Here's where everything you learned shows up:

| Concept | Where |
|---|---|
| structs, enums, `derive` (ch. 03, 06) | `models.rs`, `error.rs` |
| modules and privacy (ch. 04) | the file layout above |
| `Result`, `?`, custom errors (ch. 05) | every handler, `error.rs` |
| traits (`IntoResponse`, `Clone`, `From`) (ch. 06) | `error.rs`, `state.rs` |
| `Option` combinators, closures (ch. 03, 07) | `get_todo`, `update_todo` |
| `Arc<RwLock<T>>` (ch. 08) | `state.rs` |
| async, tokio (ch. 09) | everything |

---

## Step 0: Dependencies

From the `service-practice` folder:

```bash
cargo add tokio --features full          # (already there if you did ch. 09)
cargo add axum
cargo add serde --features derive
cargo add serde_json
cargo add thiserror
cargo add tracing
cargo add tracing-subscriber --features env-filter
cargo add tower-http --features trace
cargo add --dev tower --features util
cargo add --dev http-body-util
```

Your `Cargo.toml` should now look like this (exact patch versions may differ):

```toml
[package]
name = "service-practice"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
tokio = { version = "1", features = ["full"] }
tower-http = { version = "0.7", features = ["trace"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter"] }

[dev-dependencies]
http-body-util = "0.1"
tower = { version = "0.5", features = ["util"] }
```

What each one is for:

| Crate | Role |
|---|---|
| `tokio` | async runtime (ch. 09) |
| `axum` | web framework: routing, extractors, responses |
| `serde` + `serde_json` | convert structs ⇄ JSON via `#[derive(Serialize, Deserialize)]` |
| `thiserror` | derive `std::error::Error` + `Display` for our error enum |
| `tracing` + `tracing-subscriber` | structured logging |
| `tower-http` | ready-made HTTP middleware (request logging) |
| `tower`, `http-body-util` (dev) | call the app in tests without a real network |

Until step 5 you'll see `warning: unused dependency` for crates you haven't used
yet. That's expected, and the warnings disappear once the code uses them.

---

## Step 1: Hello, axum

Replace `src/main.rs` with:

```rust
// file: src/main.rs (step 1)
use axum::{Router, routing::get};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("listening on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}

async fn hello() -> &'static str {
    "Hello from Rust!"
}
```

```bash
cargo run
# in another terminal:
curl localhost:3000
```

🎉 **That's your first running Rust service.**

What's happening:

- `Router::new().route("/", get(hello))` means "for `GET /`, call `hello`".
- A **handler** is just an `async fn`. Its return type must implement axum's
  `IntoResponse` trait. `&'static str`, `String`, `Json<T>`, `StatusCode`, tuples
  like `(StatusCode, Json<T>)`, and `Result<T, E>` (where both sides implement
  `IntoResponse`) all do.
- `axum::serve` runs the accept loop. Every request becomes a tokio task, as in the
  echo server in chapter 09.

---

## Step 2: Data models and JSON

Create `src/models.rs`:

```rust
// file: src/models.rs
use serde::{Deserialize, Serialize};

/// A todo item as stored and returned by the API.
#[derive(Debug, Clone, Serialize)]
pub struct Todo {
    pub id: u64,
    pub title: String,
    pub done: bool,
}

/// Body of `POST /todos`.
#[derive(Debug, Deserialize)]
pub struct CreateTodo {
    pub title: String,
}

/// Body of `PATCH /todos/{id}`: every field is optional.
#[derive(Debug, Deserialize)]
pub struct UpdateTodo {
    pub title: Option<String>,
    pub done: Option<bool>,
}
```

- `Serialize` means "can become JSON" (for responses). `Deserialize` means "can be
  built from JSON" (for request bodies). serde generates that code at compile time.
- `Option<T>` fields: if the key is missing from the JSON, you get `None`. That's
  exactly what "partial update" means. The type models the domain (ch. 03).
- `Clone` on `Todo`: we'll hand out copies of todos while the original stays in
  the store.
- We use separate input types (`CreateTodo`) and output types (`Todo`), so a client
  can't set `id` or `done` when creating a todo. Invalid input can't even be
  represented.

---

## Step 3: Shared state + list and create

Create `src/state.rs`:

```rust
// file: src/state.rs
use std::{
    collections::HashMap,
    sync::{Arc, RwLock},
};

use crate::models::Todo;

#[derive(Default)]
pub struct Db {
    pub todos: HashMap<u64, Todo>,
    pub next_id: u64,
}

/// Cheap to clone: cloning only bumps the Arc's reference count.
#[derive(Clone, Default)]
pub struct AppState {
    pub db: Arc<RwLock<Db>>,
}
```

Why `Arc<RwLock<Db>>`? (This is chapter 08.)

- Requests run **concurrently** on several threads, and they all need the same `Db`.
- `Arc`: shared ownership across threads. axum clones `AppState` for each request.
  That clone is cheap: only the reference count changes, the data isn't copied.
- `RwLock`: many concurrent readers (`GET`), or one writer (`POST`/`PATCH`/`DELETE`).
- `#[derive(Default)]` gives us `AppState::default()` with an empty map and
  `next_id = 0`.

Create `src/routes.rs`:

```rust
// file: src/routes.rs (step 3)
use axum::{
    Json, Router,
    extract::State,
    http::StatusCode,
    routing::get,
};
use serde_json::{Value, json};

use crate::{
    models::{CreateTodo, Todo},
    state::AppState,
};

pub fn router(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/todos", get(list_todos).post(create_todo))
        .with_state(state)
}

async fn health() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}

async fn list_todos(State(state): State<AppState>) -> Json<Vec<Todo>> {
    let db = state.db.read().unwrap();
    let mut todos: Vec<Todo> = db.todos.values().cloned().collect();
    todos.sort_by_key(|t| t.id);
    Json(todos)
}

async fn create_todo(
    State(state): State<AppState>,
    Json(input): Json<CreateTodo>,
) -> (StatusCode, Json<Todo>) {
    let mut db = state.db.write().unwrap();
    db.next_id += 1;
    let todo = Todo {
        id: db.next_id,
        title: input.title,
        done: false,
    };
    db.todos.insert(todo.id, todo.clone());
    (StatusCode::CREATED, Json(todo))
}
```

And update `src/main.rs`:

```rust
// file: src/main.rs (step 3)
mod models;
mod routes;
mod state;

use crate::state::AppState;

#[tokio::main]
async fn main() {
    let app = routes::router(AppState::default());

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("listening on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

Run it and try it:

```bash
cargo run
curl localhost:3000/health
curl -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"title":"learn ownership"}'
curl -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"title":"learn traits"}'
curl localhost:3000/todos
curl -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"wrong":1}'   # automatic 422 error
```

### Extractors: the core idea of axum ⭐

Look at the handler parameters:

```rust
async fn create_todo(State(state): State<AppState>, Json(input): Json<CreateTodo>)
```

Each parameter is an **extractor**: a type that knows how to pull something out of
the request. axum calls them before your handler runs:

| Extractor | Gets |
|---|---|
| `State<T>` | the shared state passed to `.with_state(...)` |
| `Json<T>` | the body, parsed as JSON into `T` (rejects bad JSON automatically) |
| `Path<T>` | URL parameters like `{id}` |
| `Query<T>` | `?page=2&limit=10` |
| `HeaderMap` | request headers |

`State(state): State<AppState>` is **pattern destructuring in a parameter** (ch. 03).
`State` is a tuple struct, and we pull out the inner value directly.

Rule: the extractor that consumes the **body** (`Json`) must be the **last**
parameter.

Notice the lock scope: `let db = state.db.read().unwrap();` returns a guard that's
dropped when the function returns, and the lock is released then (RAII, ch. 08).
We never hold it across an `.await`, so `std::sync::RwLock` is the right choice
(ch. 09).

---

## Step 4: Errors as HTTP responses

Right now an empty title is accepted, and there's no way to say "404 not found".
Let's model errors as an enum (ch. 05) and teach axum how to turn it into a
response (a trait impl, ch. 06).

Create `src/error.rs`:

```rust
// file: src/error.rs
use axum::{
    Json,
    http::StatusCode,
    response::{IntoResponse, Response},
};
use serde_json::json;

#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("todo {0} not found")]
    NotFound(u64),

    #[error("invalid input: {0}")]
    BadRequest(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let status = match &self {
            AppError::NotFound(_) => StatusCode::NOT_FOUND,
            AppError::BadRequest(_) => StatusCode::BAD_REQUEST,
        };
        let body = Json(json!({ "error": self.to_string() }));
        (status, body).into_response()
    }
}
```

- `#[derive(thiserror::Error)]` + `#[error("...")]` generates the `Display` and
  `Error` impls you wrote by hand in chapter 05. `{0}` refers to the variant's
  first field.
- `impl IntoResponse for AppError` means handlers can now return
  `Result<T, AppError>`, and with `?` an error becomes a clean JSON response like
  `{"error": "todo 42 not found"}` with the right status code.

Now replace `src/routes.rs` with the full version: all CRUD routes plus validation.

```rust
// file: src/routes.rs (step 4)
use axum::{
    Json, Router,
    extract::{Path, State},
    http::StatusCode,
    routing::get,
};
use serde_json::{Value, json};

use crate::{
    error::AppError,
    models::{CreateTodo, Todo, UpdateTodo},
    state::AppState,
};

pub fn router(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/todos", get(list_todos).post(create_todo))
        .route(
            "/todos/{id}",
            get(get_todo).patch(update_todo).delete(delete_todo),
        )
        .with_state(state)
}

async fn health() -> Json<Value> {
    Json(json!({ "status": "ok" }))
}

async fn list_todos(State(state): State<AppState>) -> Json<Vec<Todo>> {
    let db = state.db.read().unwrap();
    let mut todos: Vec<Todo> = db.todos.values().cloned().collect();
    todos.sort_by_key(|t| t.id);
    Json(todos)
}

async fn create_todo(
    State(state): State<AppState>,
    Json(input): Json<CreateTodo>,
) -> Result<(StatusCode, Json<Todo>), AppError> {
    let title = input.title.trim();
    if title.is_empty() {
        return Err(AppError::BadRequest("title must not be empty".into()));
    }

    let mut db = state.db.write().unwrap();
    db.next_id += 1;
    let todo = Todo {
        id: db.next_id,
        title: title.to_string(),
        done: false,
    };
    db.todos.insert(todo.id, todo.clone());
    tracing::info!(id = todo.id, "created todo");

    Ok((StatusCode::CREATED, Json(todo)))
}

async fn get_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Todo>, AppError> {
    let db = state.db.read().unwrap();
    db.todos
        .get(&id)
        .cloned()
        .map(Json)
        .ok_or(AppError::NotFound(id))
}

async fn update_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(input): Json<UpdateTodo>,
) -> Result<Json<Todo>, AppError> {
    let mut db = state.db.write().unwrap();
    let todo = db.todos.get_mut(&id).ok_or(AppError::NotFound(id))?;

    if let Some(title) = input.title {
        todo.title = title;
    }
    if let Some(done) = input.done {
        todo.done = done;
    }
    Ok(Json(todo.clone()))
}

async fn delete_todo(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<StatusCode, AppError> {
    let mut db = state.db.write().unwrap();
    db.todos.remove(&id).ok_or(AppError::NotFound(id))?;
    Ok(StatusCode::NO_CONTENT)
}
```

Add `mod error;` to the top of `src/main.rs` (next to the other `mod` lines), then
`cargo run`.

Read `get_todo` slowly. It's a whole pipeline built from `Option` combinators
(ch. 07):

```rust
db.todos
    .get(&id)                    // Option<&Todo>        — borrow from the map
    .cloned()                    // Option<Todo>         — own a copy (the lock guard dies at return)
    .map(Json)                   // Option<Json<Todo>>   — a tuple-struct constructor is a function!
    .ok_or(AppError::NotFound(id)) // Result<Json<Todo>, AppError>
```

Why `.cloned()`? `get` returns a reference **into the locked map**. The guard is
dropped when the function returns, so the reference can't escape. That's the
borrow checker preventing a use-after-unlock bug. Cloning hands the client its own
copy.

In `update_todo`, `get_mut` gives a `&mut Todo` into the map, so we modify it **in
place**. The `?` returns a 404 early if the id doesn't exist.

Note for older tutorials: axum 0.8 writes path parameters as `/{id}`. Older
versions used `/:id`.

Try the errors:

```bash
curl -i localhost:3000/todos/42                     # 404 {"error":"todo 42 not found"}
curl -i -X POST localhost:3000/todos -H 'content-type: application/json' -d '{"title":"  "}'  # 400
curl -X PATCH localhost:3000/todos/1 -H 'content-type: application/json' -d '{"done":true}'
curl -i -X DELETE localhost:3000/todos/1            # 204
```

---

## Step 5: Logging, configuration, graceful shutdown

A production-style `main`: structured logs, the port taken from an environment
variable, an HTTP request-logging middleware, and a clean stop on Ctrl+C.

Final `src/main.rs`:

```rust
// file: src/main.rs
mod error;
mod models;
mod routes;
mod state;

use tower_http::trace::TraceLayer;
use tracing_subscriber::EnvFilter;

use crate::state::AppState;

#[tokio::main]
async fn main() {
    // Logging: RUST_LOG=debug cargo run   (defaults to "info")
    tracing_subscriber::fmt()
        .with_env_filter(
            EnvFilter::try_from_default_env().unwrap_or_else(|_| EnvFilter::new("info")),
        )
        .init();

    let port = std::env::var("PORT").unwrap_or_else(|_| "3000".to_string());
    let addr = format!("0.0.0.0:{port}");

    let app = routes::router(AppState::default()).layer(TraceLayer::new_for_http());

    let listener = tokio::net::TcpListener::bind(&addr)
        .await
        .expect("failed to bind address");
    tracing::info!("listening on http://{addr}");

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .expect("server error");
}

async fn shutdown_signal() {
    tokio::signal::ctrl_c()
        .await
        .expect("failed to install Ctrl+C handler");
    tracing::info!("shutting down");
}
```

```bash
cargo run                                          # info logs
RUST_LOG=debug cargo run                           # see per-request logs from TraceLayer
RUST_LOG=service_practice=debug,tower_http=debug cargo run
PORT=8080 cargo run                                # different port
```

- `tracing::info!(id = todo.id, "created todo")` is **structured** logging: `id` is
  a field, not just text, so log tools can filter on it.
- `.layer(TraceLayer::new_for_http())` wraps the whole router in middleware (a
  tower `Layer`) that logs each request and its response time.
- `.with_graceful_shutdown(...)` stops accepting new connections when the future
  completes (on Ctrl+C), and lets in-flight requests finish first.
- `expect("...")` at startup: if we can't bind the port, crashing immediately with a
  clear message is the right behavior.

---

## Step 6: Tests

axum's `Router` is a tower `Service`, so tests can send it requests **in memory**:
no port, no network, and fast.

Append this to the bottom of `src/routes.rs`:

```rust
// file: src/routes.rs (step 6, append)
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::Request;
    use http_body_util::BodyExt;
    use tower::ServiceExt; // for `.oneshot()`

    fn app() -> Router {
        router(AppState::default())
    }

    #[tokio::test]
    async fn health_returns_ok() {
        let res = app()
            .oneshot(Request::get("/health").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::OK);
    }

    #[tokio::test]
    async fn create_then_get() {
        let app = app();

        let res = app
            .clone()
            .oneshot(
                Request::post("/todos")
                    .header("content-type", "application/json")
                    .body(Body::from(r#"{"title":"learn rust"}"#))
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::CREATED);

        let bytes = res.into_body().collect().await.unwrap().to_bytes();
        let created: Value = serde_json::from_slice(&bytes).unwrap();
        assert_eq!(created["title"], "learn rust");

        let res = app
            .oneshot(Request::get("/todos/1").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::OK);
    }

    #[tokio::test]
    async fn missing_todo_is_404() {
        let res = app()
            .oneshot(Request::get("/todos/999").body(Body::empty()).unwrap())
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::NOT_FOUND);
    }

    #[tokio::test]
    async fn empty_title_is_400() {
        let res = app()
            .oneshot(
                Request::post("/todos")
                    .header("content-type", "application/json")
                    .body(Body::from(r#"{"title":"   "}"#))
                    .unwrap(),
            )
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::BAD_REQUEST);
    }
}
```

```bash
cargo test
```

- `#[tokio::test]` is like `#[test]`, but gives each test an async runtime.
- `app.clone()`: `oneshot` consumes the router (it takes `self`), so we clone it to
  send a second request. The clones share the same `Arc` state, so the todo we
  created is still there.
- `r#"..."#` is a **raw string**: no need to escape the `"` characters.

---

## Step 7: Ship it

```bash
cargo clippy            # fix any warnings it shows
cargo fmt
cargo build --release
./target/release/service-practice
```

The release binary is a **single self-contained executable**. Copy it to a Linux
server and run it: no runtime and no VM. It typically starts in milliseconds and
idles at a few MB of RAM.

---

## ✅ Exercises (each one teaches something new)

1. **Query parameters:** add `GET /todos?done=true` filtering. Hints: a
   `#[derive(Deserialize)] struct ListParams { done: Option<bool> }` and an
   `axum::extract::Query<ListParams>` extractor, then `.filter(...)` in
   `list_todos`.
2. **Validation:** reject titles longer than 200 characters with
   `AppError::BadRequest`. Apply the same rule in `update_todo`. Then refactor it
   into one `fn validate_title(s: &str) -> Result<String, AppError>` and use `?`.
3. **Timestamps:** add a `created_at` field. Use `cargo add chrono --features serde`
   and the type `chrono::DateTime<chrono::Utc>`.
4. **A new error variant:** add `AppError::Conflict(String)` (409) and reject
   creating a todo whose title already exists. Notice how the compiler forces you
   to update the `match` in `into_response`.
5. **Integration tests:** move the code into a library (`src/lib.rs` with `pub mod`
   for each module, plus `pub fn router`) and write a test in `tests/api.rs` that
   only uses the public API (ch. 04).
6. **Real database (big step):** replace the `HashMap` with SQLite using
   [`sqlx`](https://docs.rs/sqlx). Handlers become `sqlx::query_as!(...).fetch_all(&pool).await?`,
   `AppState` holds a `sqlx::SqlitePool` (which is already cheap to clone), and you
   add `#[error(transparent)] Db(#[from] sqlx::Error)` to `AppError` so that `?`
   converts database errors automatically. That's the same `From` mechanism as in
   ch. 05.

<details>
<summary>Hint for exercise 1</summary>

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Debug, Deserialize)]
pub struct ListParams {
    pub done: Option<bool>,
}

async fn list_todos(
    State(state): State<AppState>,
    Query(params): Query<ListParams>,
) -> Json<Vec<Todo>> {
    let db = state.db.read().unwrap();
    let mut todos: Vec<Todo> = db
        .todos
        .values()
        .filter(|t| params.done.is_none_or(|d| t.done == d))
        .cloned()
        .collect();
    todos.sort_by_key(|t| t.id);
    Json(todos)
}
```

</details>

---

## Where to go next

- **The Book, ch. 18–20** (OOP patterns, advanced patterns, `unsafe`, advanced traits, macros)
- **The Book, ch. 21**: build a multithreaded web server from scratch with only `std`
- **Rustlings** (https://rustlings.rust-lang.org/): keep your syntax sharp
- **Zero To Production In Rust** by Luca Palmieri: a whole book on building a
  production Rust API (community book, not free, but highly regarded)
- **axum examples** (https://github.com/tokio-rs/axum/tree/main/examples): auth, websockets, databases, and more
- **Jon Gjengset's "Crust of Rust"** videos: deep dives into iterators, lifetimes, smart pointers, async

You now know how Rust works **and** you have a running service. 🦀
