# Style, lints, workspace, profile

## `rustfmt.toml`

**The canonical config lives in `assets/rustfmt.toml`.** Treat that file as the source of truth — copy it into any project that doesn't have a `rustfmt.toml` (see "Project bootstrap" in SKILL.md).

Highlights of the canonical config:

- `edition = "2024"`
- `max_width = 100`, `tab_spaces = 4`, no hard tabs
- `imports_granularity = "Crate"` — collapses `use std::a; use std::b;` → `use std::{a, b};`
- `group_imports = "StdExternalCrate"` — orders std → external → crate, separated by blank lines
- `wrap_comments`, `format_code_in_doc_comments`, `normalize_comments`, `normalize_doc_attributes` — keep docs tidy
- `chain_width = 60`, `fn_call_width = 60`, `single_line_if_else_max_width = 50`
- `trailing_comma = "Vertical"`, `array_width = 60`, `struct_lit_width = 18`, `struct_variant_width = 35`
- `control_brace_style = "AlwaysSameLine"` — `if … {` on one line
- `match_arm_blocks = true`, `match_block_trailing_comma = false`
- `newline_style = "Unix"`, `blank_lines_upper_bound = 2`

Many of these are nightly rustfmt features (`format_code_in_doc_comments`, `normalize_*`, `wrap_comments`) — fine on this user's nightly toolchain.

## Pre-commit workflow

Outlined at top level in SKILL.md ("Pre-commit workflow"). Repeated here as a recipe with rationale:

```bash
# Step 1 — clippy with auto-fix
cargo clippy --fix --allow-dirty --allow-staged --all-targets --all-features
```

- `--fix` applies machine-applicable lint fixes in place.
- `--allow-dirty --allow-staged` tells clippy to operate on a working tree that already has changes (the very changes you're about to commit). Without these flags, clippy refuses to touch anything.
- `--all-targets` — includes `tests/`, `benches/`, `examples/`.
- `--all-features` — lints feature-gated code too.

```bash
# Step 2 — format
cargo fmt --all
```

- `--all` formats every crate in the workspace.
- rustfmt config comes from `rustfmt.toml` at the workspace root (seeded by Project bootstrap if missing).

```bash
# Step 3 — stage + commit
git add <specific paths>
git commit -m "..."
```

- Use specific paths, not `git add -A` — the auto-fix may have touched files outside the original change scope; review those separately.
- If clippy/fmt left more than one logical change unstaged, consider splitting into multiple commits.

### When to skip

The workflow targets `.rs` source. Skip it when the commit only touches:

- Markdown / docs files.
- `Cargo.toml` metadata-only edits (license/description/url).
- `.github/workflows/` CI files.

If `.rs` files are in the changeset at all, run the full workflow.

### What clippy --fix does NOT cover

- Logic bugs (always manual).
- Lints marked `MachineApplicable = false` (require human judgment).
- `clippy::pedantic` / `clippy::nursery` lints that touch semantics.

After auto-fix, re-run `cargo clippy --all-targets --all-features -- -D warnings` to see what's left. Treat remaining warnings as a failure to commit unless documented.

## Crate-level lints (in `lib.rs` / `main.rs`)

```rust
#![warn(
    clippy::pedantic,
    clippy::nursery,
    clippy::cargo,
    unsafe_op_in_unsafe_fn,
    rust_2024_compatibility,
    missing_docs,                        // for libraries; remove for binaries
    rustdoc::broken_intra_doc_links,
)]
#![allow(
    clippy::module_name_repetitions,     // common in this codebase
    clippy::must_use_candidate,          // noisy
    clippy::missing_errors_doc,          // for binaries only
    clippy::missing_panics_doc,          // for binaries only
)]
```

## `clippy.toml`

The `disallowed_types` lint enforces the std-first policy in code:

```toml
disallowed-types = [
    { path = "parking_lot::Mutex",       reason = "use std::sync::Mutex" },
    { path = "parking_lot::RwLock",      reason = "use std::sync::RwLock" },
    { path = "ahash::AHashMap",          reason = "use std::collections::HashMap" },
    { path = "smallvec::SmallVec",       reason = "use Vec or [T; N]" },
    { path = "arrayvec::ArrayVec",       reason = "use [T; N] with const generics" },
    { path = "once_cell::sync::Lazy",    reason = "use std::sync::LazyLock (stable 1.80)" },
    { path = "once_cell::sync::OnceCell", reason = "use std::sync::OnceLock" },
    { path = "lazy_static::lazy_static", reason = "use std::sync::LazyLock" },
    { path = "chrono::DateTime",         reason = "use jiff::Zoned (calendar) or std::time::SystemTime" },
]

disallowed-methods = [
    { path = "std::env::args",           reason = "for non-trivial CLI use clap derive" },
]
```

Run with `cargo clippy -- -D warnings` in CI.

## Workspace layout

Standard:

```
my-project/
├── Cargo.toml              # workspace root
├── rust-toolchain.toml
├── rustfmt.toml
├── clippy.toml
├── deny.toml               # cargo-deny config
├── .cargo/config.toml      # optional: target-specific build flags
├── crates/
│   ├── core/
│   ├── api/
│   └── cli/
├── xtask/                  # build tasks (not Makefile)
└── tests/                  # workspace-level integration tests
```

## `Cargo.toml` (workspace root)

```toml
[workspace]
members = ["crates/*", "xtask"]
default-members = ["crates/*"]   # excludes xtask from `cargo build`
resolver = "3"

[workspace.package]
edition = "2024"
license = "MIT OR Apache-2.0"
rust-version = "1.85"
repository = "https://github.com/user/repo"

[workspace.dependencies]
# Approved crates. Default to latest published version.
# Switch any entry to git = "..." with rev = "<sha>" when GitHub master is
# meaningfully ahead of the latest release. See SKILL.md "Versioning and
# source-of-truth" for the policy.
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }
clap = { version = "4", features = ["derive"] }
regex = "1"
rayon = "1"
bytemuck = { version = "1", features = ["derive"] }
bytes = "1"
num-traits = "0.2"
jiff = "0.1"

# Dev-deps
criterion = "0.5"
proptest = "1"
insta = "1"
tempfile = "3"

# Example of a git-tracking dependency, when needed (default to branch):
# tokio = { git = "https://github.com/tokio-rs/tokio", branch = "master", features = ["full"] }
# Use rev = "<full-sha>" only for crates with a track record of breaking APIs on master.

[profile.dev]
opt-level = 1

[profile.release]
lto = "thin"
codegen-units = 1
panic = "abort"
strip = "symbols"

[profile.bench]
inherits = "release"
debug = true
```

## `deny.toml` (cargo-deny)

```toml
[advisories]
yanked = "deny"
ignore = []

[bans]
multiple-versions = "warn"
deny = [
    { name = "anyhow" },
    { name = "thiserror" },
    { name = "parking_lot" },
    { name = "ahash" },
    { name = "smallvec" },
    { name = "arrayvec" },
    { name = "async-trait" },
    { name = "chrono" },
    { name = "time" },
    { name = "log" },
    { name = "once_cell" },
    { name = "lazy_static" },
]

[licenses]
allow = ["MIT", "Apache-2.0", "BSD-3-Clause", "Unicode-DFS-2016", "ISC"]
confidence-threshold = 0.9
```

Run `cargo deny check` in CI.

## `xtask` pattern

For build tasks beyond plain `cargo build` (codegen, packaging, custom workflows), use the [xtask pattern](https://github.com/matklad/cargo-xtask): a regular Rust crate at `xtask/` invoked as `cargo xtask <subcommand>`. Don't use Makefiles or shell scripts for non-trivial build logic.
