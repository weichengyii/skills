# Error handling

## Top-level rule

- **Binary / app crate**: `Result<T, Box<dyn std::error::Error + Send + Sync>>` is the default return type for `main` and orchestration code. Errors propagate via `?`.
- **Library crate**: hand-write a `#[non_exhaustive] enum LibError` per crate, `impl std::error::Error` + `impl Display`. **No `thiserror` derive macro.** No `anyhow`.

## Hand-written `impl Error` template

Copy this 30-line skeleton when defining a new library error type.

```rust
use std::fmt;

#[derive(Debug)]
#[non_exhaustive]
pub enum LibError {
    Io(std::io::Error),
    Parse { line: usize, msg: String },
    OutOfRange { value: i64, max: i64 },
    // Add variants. Keep variant names noun-like ("Io", not "IoError").
}

impl fmt::Display for LibError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            Self::Io(e) => write!(f, "I/O error: {e}"),
            Self::Parse { line, msg } => write!(f, "parse error at line {line}: {msg}"),
            Self::OutOfRange { value, max } => write!(f, "value {value} exceeds max {max}"),
        }
    }
}

impl std::error::Error for LibError {
    fn source(&self) -> Option<&(dyn std::error::Error + 'static)> {
        match self {
            Self::Io(e) => Some(e),
            _ => None,
        }
    }
}

impl From<std::io::Error> for LibError {
    fn from(e: std::io::Error) -> Self {
        Self::Io(e)
    }
}
```

Total: 30 lines. Comparable `thiserror` derive would be 12 lines of macro attributes — saved ~18 lines for one external proc-macro dependency. Not worth it.

## When to add `From` impls

One `From` impl per upstream error type that flows through `?`. Don't add `From<String>` or `From<&str>` — that's the `anyhow::Context` anti-pattern that loses type information.

## Adding context to errors

The `anyhow::Context` pattern looks like `result.context("loading config")?`. Replace it with `.map_err(|e| MyError::ConfigLoad(e.to_string()))` — keep the typed variant, accept the slightly more verbose call site.

Or, for one-line context in a `Box<dyn Error>` context (binary crate):

```rust
fn run() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let cfg = load_config()
        .map_err(|e| format!("loading config: {e}"))?;
    // ...
}
```

## Backtrace

`std::backtrace::Backtrace` is stable. Attach it to error variants that benefit:

```rust
use std::backtrace::Backtrace;

#[derive(Debug)]
pub enum LibError {
    Panic { backtrace: Backtrace, msg: String },
}

// Capture at error-creation time:
LibError::Panic {
    backtrace: Backtrace::capture(),  // only captures if RUST_BACKTRACE=1
    msg: "unexpected state".into(),
}
```

`Backtrace::capture()` is cheap when `RUST_BACKTRACE` isn't set (returns `Disabled`). No reason to skip it for hot paths in error code.

## Panic policy

Use `panic!` / `unreachable!()` / `unwrap()` only when:

1. **Invariant violation in your own code** — something is structurally broken (e.g. `slice[i]` where `i` was just bounds-checked locally).
2. **Pre-startup configuration is invalid** — `main()` can panic on bad config before any work starts.
3. **Programmer error in tests** — `unwrap()` in tests is fine.

For everything user-facing or runtime-recoverable, use `Result`.

When you do use `unwrap`/`expect`, **prefer `expect` with a clear message** explaining why it's safe:

```rust
let header = bytes.get(..8).expect("len ≥ 8 checked above");
```

## Async errors

In async code under `tokio`, prefer `Result<T, LibError>` where `LibError` has variants for cancellation-related failures (`tokio::time::error::Elapsed`, `tokio::sync::mpsc::error::SendError`, etc.). Don't swallow these — surface them.
