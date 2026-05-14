# Async (tokio)

## Runtime

Use `tokio` multi-threaded runtime (`#[tokio::main]` or explicit `tokio::runtime::Builder::new_multi_thread()`). Single-threaded only when you have a specific reason (e.g. embedded, deterministic testing).

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    tracing_subscriber::fmt()
        .with_env_filter(tracing_subscriber::EnvFilter::from_default_env())
        .init();
    run().await
}
```

## Async fn in trait — use native, not `async-trait`

Native `async fn` in trait is stable since 1.75 and works in edition 2024 without ceremony:

```rust
pub trait Service {
    async fn handle(&self, req: Request) -> Result<Response, ServiceError>;
}
```

The `async-trait` crate is forbidden — it boxes futures unconditionally and adds a dependency for no benefit on this user's toolchain.

**Send caveat**: native `async fn` in trait does *not* guarantee `Send` on the returned future. If you need `Send`, use the explicit form:

```rust
pub trait Service {
    fn handle(&self, req: Request) -> impl Future<Output = Result<Response, ServiceError>> + Send;
}
```

Or, with `return_type_notation` (nightly), constrain at the use site.

## Sync primitives inside async

| Sync vs Async | Use |
|---|---|
| **Short critical section** (no `.await` inside the lock) | `std::sync::Mutex` / `RwLock` — faster, no async-aware overhead |
| **Long critical section** or holding lock across `.await` | `tokio::sync::Mutex` / `RwLock` — cancellation-safe |
| One-shot signal | `tokio::sync::oneshot` |
| Multi-producer queue | `tokio::sync::mpsc` |
| Broadcast | `tokio::sync::broadcast` |
| Notification | `tokio::sync::Notify` |

**Default to `std::sync` for locks**. Reach for `tokio::sync::Mutex` only when you genuinely need to hold the lock across `.await` (uncommon — usually a redesign).

## Blocking work in async

Never call blocking code (file I/O, CPU-bound loops, sync mutex contended for >µs) directly inside an async function. Wrap in `tokio::task::spawn_blocking`:

```rust
let parsed = tokio::task::spawn_blocking(move || {
    parse_huge_file_sync(&path)
})
.await??;  // first ? = JoinError, second ? = inner parse error
```

For CPU-bound parallel work over a collection, use `rayon::par_iter` inside `spawn_blocking`:

```rust
tokio::task::spawn_blocking(move || {
    use rayon::prelude::*;
    items.par_iter().map(process).collect::<Vec<_>>()
}).await?
```

## Cancellation

`tokio` cancels by dropping the future. Code that holds external resources must implement Drop or use `tokio::select!` with `tokio::pin!` for explicit cleanup paths.

Pattern: `select!` with a `shutdown` signal:

```rust
tokio::select! {
    biased;
    _ = shutdown.notified() => { /* cleanup */ Ok(()) }
    result = work() => result,
}
```

`biased;` ensures the shutdown branch is checked first (without it, `select!` picks pseudo-randomly).

## Channels and backpressure

Default `tokio::sync::mpsc` with a **bounded** channel. Unbounded channels are a memory-leak pit:

```rust
let (tx, mut rx) = tokio::sync::mpsc::channel::<Event>(1024);  // bounded
```

Backpressure: `tx.send().await` waits when full. This is the correct behavior — propagate the slowness upstream.

## Tracing in async

Every async fn that does meaningful work gets `#[tracing::instrument]`. Fields capture the inputs:

```rust
#[tracing::instrument(skip(self), fields(user_id = %req.user_id))]
async fn handle(&self, req: Request) -> Result<Response, ServiceError> {
    // ...
}
```

`skip(self)` keeps spans readable. `%` formats with `Display`; use `?` for `Debug`.
