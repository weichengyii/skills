# Toolchain & language features

## `rust-toolchain.toml`

Commit this at the workspace root:

```toml
[toolchain]
channel = "nightly"
components = ["rustfmt", "clippy", "rust-src", "rust-analyzer", "miri", "llvm-tools"]
profile = "default"
```

- **`channel = "nightly"`** — rolling. Don't pin a date unless a regression forces it; the user wants to track features as they land.
- **`components`** — `rust-src` needed for `std::simd` and certain clippy lints; `miri` for unsafe verification; `llvm-tools` for `cargo-llvm-cov`.

## Edition

Always **`edition = "2024"`** in every `Cargo.toml`. Use `[workspace.package]` to set it once for a workspace:

```toml
[workspace.package]
edition = "2024"
rust-version = "1.85"  # or whatever minimum the unstable features require
```

(`rust-version` documents intent; doesn't gate anything on nightly.)

## Unstable features

Three categories — white, grey, black.

### White (default-on; turn on as needed without ceremony)

| Feature | Why |
|---|---|
| `generic_const_exprs` | Arithmetic on const generics (`N + M`, `N / 2`). Marked incomplete, but stable enough in practice. Add `#![allow(incomplete_features)]` alongside |
| `type_alias_impl_trait` / `impl_trait_in_assoc_type` | Named existential types — needed for sane state-machine encoding |
| `let_chains` | `if let Some(x) = a && x > 0` — strictly more readable than nesting |
| `inline_const` | `const { … }` blocks at expression position — assert!s and compile-time checks |
| `try_blocks` | `try { … }` short-circuit groups, especially useful in tests |
| `never_type` | `!` type usable in match arms, type position, etc. |
| `negative_impls` | Express `!Send` / `!Sync` explicitly when needed |
| `portable_simd` (`std::simd`) | Default SIMD entry point — preferred over hand-rolled intrinsics |
| `min_specialization` | Limited specialization without soundness hole; use over full `specialization` |
| `async_iterator` | Stream trait in std::async_iter — use over `futures::Stream` when possible |
| `const_trait_impl` | `const fn` calls into trait methods — needed for compile-time math via traits |

Add at crate root (`lib.rs` / `main.rs`):

```rust
#![feature(
    generic_const_exprs,
    type_alias_impl_trait,
    let_chains,
    inline_const,
    try_blocks,
    never_type,
    portable_simd,
    min_specialization,
)]
#![allow(incomplete_features)]  // for generic_const_exprs
```

### Grey (use with awareness — API may shift)

- `coroutines` / `generators` — useful but API still churning. OK to use; expect refactors.
- `unboxed_closures` + `fn_traits` — only when implementing `Fn`/`FnMut`/`FnOnce` by hand (rare).

### Black (forbidden)

| Feature | Why |
|---|---|
| `specialization` (full) | Known soundness hole. Use `min_specialization` instead. |
| `const_generics_defaults` with non-trivial expressions on **non-last** parameters | Buggy currently; keep defaults last in the parameter list. |

## Required workspace `Cargo.toml` chrome

```toml
[workspace]
members = ["crates/*"]
resolver = "3"  # edition 2024 default

[workspace.package]
edition = "2024"
license = "MIT OR Apache-2.0"   # or whatever the project picks
rust-version = "1.85"

[workspace.dependencies]
# Centralized version pinning. Member crates use `tokio = { workspace = true }`.
# Pin only approved crates (see crates-whitelist.md).
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
clap = { version = "4", features = ["derive"] }

[profile.dev]
opt-level = 1     # debug builds are faster to run; small comp-time hit

[profile.release]
lto = "thin"
codegen-units = 1
panic = "abort"
strip = "symbols"

[profile.bench]
inherits = "release"
debug = true       # so flamegraphs / perf can attribute samples
```

## Reading new feature stabilization

The user tracks the [Rust unstable book](https://doc.rust-lang.org/nightly/unstable-book/) and what stabilizes each release. When a whitelist feature stabilizes:
- Remove it from the `#![feature(...)]` list.
- No other code change needed (semantics don't change at stabilization).
