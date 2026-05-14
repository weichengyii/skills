# Crates whitelist — authoritative per-crate rationale

Open this file whenever a crate decision needs justification, or whenever the user asks "should I use X?". The two lists below override generic Rust advice.

## Forbidden by default

For each: the **std replacement** is what you write *instead*, and the **why** explains the chain of reasoning. If the user pushes back with a specific contention, only then revisit.

### `anyhow`
- **std replacement**: `Box<dyn std::error::Error + Send + Sync>` for app/binary code; `?` for propagation; `std::error::Error::source()` for chaining.
- **Why**: `std::error::Error` is a stable trait. `Box<dyn Error + Send + Sync>` is `?`-compatible from any error. The only real `anyhow` value-add (`Context::context` for adding messages) is a one-line helper you can write once: `.map_err(|e| format!("...: {e}"))`. Long-term std is irreplaceable; anyhow has a single maintainer.

### `thiserror`
- **std replacement**: Hand-write `enum YourError { ... }` + `impl std::error::Error for YourError {}` + `impl Display for YourError`. See `error-handling.md` for a 12-line template you can copy.
- **Why**: `thiserror` is a proc-macro that generates `impl Display + Error`. The generated code is identical to what you'd hand-write; the build cost is real (`syn` + `quote` + `proc-macro2`). For an error type with ≤8 variants, hand-writing takes ~30 seconds and avoids the dependency entirely.

### `derive_more`
- **std replacement**: Hand-write the few `impl From`, `impl Display`, etc. you actually need. Most cases are one line each.
- **Why**: Same reasoning as `thiserror` — proc-macro overhead for code you can write in seconds.

### `parking_lot`
- **std replacement**: `std::sync::Mutex` / `RwLock` / `Once`.
- **Why**: `std::sync` got significant rewrites in 1.62+ (queue-based; no longer pthread fallback on Linux). The contention gap to `parking_lot` is ~5–15 % under heavy contention, ~0 % under light contention. Real workloads are rarely lock-bound on user-space mutex perf — and when they are, the right answer is usually to redesign the lock granularity, not swap implementations.

### `ahash` / `fxhash`
- **std replacement**: `std::collections::HashMap` (uses SipHash via `RandomState`).
- **Why**: SipHash is HashDoS-safe; ahash is not (without seeded state). For non-adversarial inputs the difference is ~2× on tiny keys, smaller on larger keys. Switch only if a profiler shows hashing in the top 5 of CPU samples for your specific workload.

### `smallvec` / `arrayvec` / `tinyvec`
- **std replacement**: `Vec<T>` (heap), or `[T; N]` const-generic array for fixed-size, or `[MaybeUninit<T>; N]` + `usize len` if you really need stack-allocated grow-up-to-N. Const generics on stable + `std::mem::MaybeUninit` cover most cases.
- **Why**: `smallvec` survives because allocation-free hot paths are real, but a `[T; N]` array is allocation-free *and* zero-dependency. For variable-length-up-to-N on the stack, write it yourself in 30 lines (or use `Vec` and let the allocator inline small allocations — modern allocators are good at this).

### `flume`
- **std replacement**: `std::sync::mpsc` for sync; `tokio::sync::mpsc` for async (under the tokio whitelist exception).
- **Why**: `std::sync::mpsc` got a major rewrite in 1.67 (no longer the cluttered old impl). Flume's edge has shrunk to ~zero for most workloads.

### `async-trait`
- **std replacement**: Native `async fn` in trait (stable since 1.75).
- **Why**: This crate exists for stable < 1.75. With nightly + edition 2024, native `async fn` is strictly better — no boxing, no `Send` complications via macro.

### `chrono`
- **std replacement**: `std::time::{Instant, Duration, SystemTime}` for non-calendar work. For calendar/timezone/parsing/formatting, use `jiff` (whitelisted).
- **Why**: `chrono` has carried `time` and security CVEs for years; `time` crate splintered off as a "safer" alternative; `jiff` (by the `chrono` author, third attempt) is now the modern choice. But for measuring elapsed time / sleeping / file mtimes, `std::time` is enough — don't pull in a date library to subtract two `Instant`s.

### `time` (the crate)
- **std replacement**: Same as `chrono` — `std::time` for non-calendar; `jiff` for calendar.
- **Why**: Splintered alternative to `chrono` from the same era. `jiff` supersedes both.

### `log`
- **std replacement**: `tracing` for structured logging (under whitelist); `eprintln!` / `std::io::stderr` for trivial binaries.
- **Why**: The `log` crate is a facade with no structured-fields/spans story. `tracing` is the modern community choice and exposes `tracing-log` for compatibility if you depend on a crate that emits `log` records.

### `once_cell`
- **std replacement**: `std::sync::OnceLock` (stable 1.70), `std::sync::LazyLock` (stable 1.80), `std::cell::OnceCell` (stable 1.70).
- **Why**: `once_cell` was the proving ground for these APIs and is now obsoleted by std equivalents with identical semantics.

### `lazy_static`
- **std replacement**: `std::sync::LazyLock` (stable 1.80).
- **Why**: Same as `once_cell` — std caught up; the macro-based syntax is no longer needed.

### `itertools`
- **std replacement**: `std::iter` + `slice::chunks` / `slice::windows` / `slice::chunks_exact`. For the few combinators std lacks (e.g. `tuple_windows`, `dedup_by`), hand-write a small `impl Iterator` adapter.
- **Why**: The UX gap is medium, not large — most cases are covered by std iterator combinators plus slice methods. Hand-writing the missing adapters is straightforward and keeps the dependency count down.

### `walkdir`
- **std replacement**: `std::fs::read_dir` with a recursive helper (~10 lines).
- **Why**: Recursive directory traversal is a 10-line `fn`. `walkdir`'s extra features (follow symlinks, filter, depth limits) are easy to add inline. UX gap is medium, not worth a dependency.

### `dirs` / `directories`
- **std replacement**: `std::env::var("HOME")` / `std::env::var("XDG_CONFIG_HOME")` / `std::env::current_dir()` with platform-specific fallbacks where needed.
- **Why**: For most projects the platform-specific path logic fits in a small helper. UX gap is medium.

## Approved exceptions

For each: the **why** explains which of the three criteria justifies it.

### `tokio` (+ `tokio::sync::*`)
- **Criterion**: (a) std has no async runtime.
- **Use**: multi-threaded runtime; `tokio::sync::mpsc` / `oneshot` / `Notify` inside async contexts (only inside async — for sync code use `std::sync`).

### `serde` + `serde_json` + `serde_derive`
- **Criterion**: (a) std has no serialization.
- **Use**: default JSON / TOML (`toml` crate) / binary (`postcard`) serialization.

### `tracing` + `tracing-subscriber`
- **Criterion**: (a) std has no structured logging.
- **Use**: in any application that emits more than a few log lines, especially with async/spans. For trivial binaries (<100 LoC), `eprintln!` is still preferred.

### `clap` (derive)
- **Criterion**: (c) human-factors gap vs `std::env::args()` is large for any CLI with subcommands or `--help` text.
- **Use**: anything with ≥2 subcommands, ≥4 arguments, or `--help` output that needs to look professional. Simple 1-arg binaries: just use `std::env::args()`.

### `criterion`
- **Criterion**: (a) std has no stable bench framework (`#[bench]` is nightly-only and limited).
- **Use**: all benchmarks. Group benches in `benches/`.

### `proptest`
- **Criterion**: (a) std has no property testing.
- **Use**: property-based tests for invariants and round-trips.

### `insta`
- **Criterion**: (a) std has no snapshot testing.
- **Use**: snapshot-based assertions for complex output (Display output, error messages, generated code).

### `regex`
- **Criterion**: (a) std has no regex engine.
- **Use**: pattern matching beyond `&str::contains` / `&str::find`.

### `rayon`
- **Criterion**: (c) human-factors gap vs `std::thread::scope` is large — `par_iter` is a one-line transform vs ~20 lines of manual fork/join.
- **Use**: data-parallel iteration over collections.

### `bytemuck`
- **Criterion**: (c) safe alternative to raw `transmute` for `Pod` / `Zeroable` casts.
- **Use**: SIMD load/store, zero-copy parsing, ABI-stable struct casts.

### `bytes`
- **Criterion**: (c) zero-copy `Bytes` / `BytesMut` UX is significantly better than `Vec<u8>` slicing in networking / streaming code.
- **Use**: networking and streaming parsers. For one-shot reads, `Vec<u8>` is enough.

### `num-traits`
- **Criterion**: (c) writing generic numeric code requires `Float` / `Integer` / `One` / `Zero` traits std doesn't provide.
- **Use**: generic numeric algorithms (filters, decimation, etc.).

### `tempfile`
- **Criterion**: (c) lifetime-managed cross-platform temp files — std `env::temp_dir()` doesn't track or clean up.
- **Use**: tests, file-system tooling.

### `jiff`
- **Criterion**: (a) std has no calendar / timezone / parsing — only `Instant` / `Duration` / `SystemTime`.
- **Use**: dates, timezones, human-readable times, ISO-8601 parsing. For elapsed-time measurements stick to `std::time::Instant`.

### `mimalloc` / `jemallocator`
- **Criterion**: (b) only after benchmarking shows the system allocator is the bottleneck.
- **Use**: rare. Document the benchmark when adopting.

### Cargo tools (binaries, not deps)
- `cargo-nextest` — test runner.
- `cargo-llvm-cov` — coverage.
- `cargo-deny` — license/advisory/ban check in CI.
- `cargo-machete` — find unused deps.
- `cargo-fuzz` — fuzzing harness.

### FFI
- `bindgen` (generate Rust from C headers) and `cbindgen` (generate C headers from Rust) — only when actually doing FFI.

## When the user asks "should I use X?"

Run this decision:

1. Is `X` on the forbidden list? → Name the std replacement, apply it, mention the trade-off in one line.
2. Is `X` on the approved list? → Use it; no need to justify.
3. Neither? → Run the three-criteria gate (no std equivalent / measured perf gap / large UX gap). State which criterion applies (if any) and decide. If unclear, default to std and explain.

## Versioning — latest, git when newer than crates.io

Once a crate passes the gate above, version selection follows the second top-level rule from SKILL.md:

- **Default**: latest published version on crates.io, including `0.x` and pre-release. Don't pin to an old major.
- **Switch to `git = "..."` when**: GitHub `master`/`main` has a merged feature, bug fix, or API change you need that isn't in the latest release yet.
- **When using git deps, default to `branch = "master"`** (or `"main"`). `Cargo.lock` records the resolved commit, so a given build is reproducible. The point of a git dep is to track the moving frontier — `rev`-pinning defeats that.
- **Use `rev = "<full-sha>"` only as a defense** against upstream crates that have a history of frequently breaking commonly-used public APIs. If you've been burned by a `cargo update` churning a specific crate's interface, pin its `rev`. For everything else, branch is fine.

Example workspace `Cargo.toml` mixing the two:

```toml
[workspace.dependencies]
# Release is sufficient
serde = { version = "1", features = ["derive"] }
tracing = "0.1"

# Newer-than-release needed — default to branch:
tokio = { git = "https://github.com/tokio-rs/tokio", branch = "master", features = ["full"] }
jiff = { git = "https://github.com/BurntSushi/jiff", branch = "master" }

# Pinning rev only because this crate's public API has been churning:
volatile-crate = { git = "https://github.com/owner/repo", rev = "abc1234..." }
```

## When docs lag the code — read the source

For approved crates that move quickly (tokio, serde, jiff, tracing, criterion), docs.rs lags both the crates.io release and master. When you need the **current** behavior — especially for unstable feature flags, new API surfaces, or recent bug fixes — go straight to the source:

1. The crate's `src/lib.rs` and module roots.
2. `tests/` and `examples/` directories — usually the cleanest, freshest usage demos.
3. `cargo doc --open` from the current `Cargo.lock` resolution to read locally generated docs against the resolved version.

Don't trust a 2023 blog post for 2026 API usage. Don't trust StackOverflow for crates that have had three major versions since.
