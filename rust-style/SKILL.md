---
name: rust-style
description: 'Use whenever the user writes, edits, reviews, or designs Rust code (.rs / Cargo.toml / workspace) — this user has fixed Rust conventions. TRIGGERS (EN & ZH): "帮我写一个 Rust", "add a feature to my Rust crate", "design a Rust API", "fix this Rust code", "set up a Rust workspace", "should I use anyhow / thiserror / parking_lot / smallvec / chrono / log / once_cell", "Rust 里怎么处理 error / async / unsafe", any Rust crate decision. Policies: nightly + edition 2024, std-first (forbidden defaults: anyhow, thiserror, derive_more, parking_lot, ahash, smallvec, arrayvec, flume, async-trait, chrono, time crate, log, once_cell, lazy_static, itertools, walkdir, dirs; approved: tokio, serde, tracing, clap, criterion, proptest, insta, regex, rayon, bytemuck, bytes, num-traits, tempfile, jiff for calendar). Hand-write impl std::error::Error (no derive). Const generics for compile-time dimensions. SAFETY comment on every unsafe. SKIP pure language theory questions.'
---

# Rust Style — personal conventions

## Who this skill is for

The user writes Rust on **nightly**, uses **edition 2024**, embraces unstable features the day they land, and treats `std` as the default toolbox. Third-party crates are accepted only when `std` truly cannot do the job, or doing it in `std` is measurably wasteful. These are not generic Rust best practices — they are this user's personal authoring policies. Apply them every time you write Rust *for them*.

## Why this skill exists

Two reasons the LLM defaults fail this user:

1. **Crate inflation.** Most public Rust advice reaches for `anyhow` / `thiserror` / `parking_lot` / `lazy_static` / `chrono` reflexively, because that was the right answer in 2019–2022. By 2024–2026, `std` has caught up on most of these (`std::sync::Mutex` perf, `LazyLock`, `OnceLock`, sturdy `Error` trait, `Backtrace`, `std::simd`, edition 2024 niceties), and many of those crates exist for diminished reasons. Defaulting to them adds maintenance risk — a third-party crate can stagnate, lose its maintainer, or churn its API; `std` cannot. This user has chosen the lower-risk path.

2. **Stable-only thinking.** Most advice assumes stable Rust. This user is on nightly and will happily turn on `generic_const_exprs`, `let_chains`, `try_blocks`, `portable_simd`, `min_specialization`, `never_type`, etc. — these are productive, not toys. The skill enables that, instead of working around their absence.

## The std-first principle (top-level rule)

> **A third-party crate is justified only if at least one of these holds:**
>
> **a)** `std` has no equivalent at all (`tokio`, `serde`, `criterion`, `regex`, `tracing`).
> **b)** The `std` equivalent has been **benchmarked** and shown insufficient for the specific workload (rare; `mimalloc` / `ahash` / `parking_lot` go here only with measurements).
> **c)** The `std` equivalent has a **meaningfully worse human-factors gap** that costs significant productivity (`clap` vs hand-rolling `std::env::args()`; `rayon::par_iter` vs hand-writing `std::thread::scope` shards).

If none of the three holds, **prefer `std`**. When in doubt, default to `std`.

The (c) clause is the easiest to abuse — every developer thinks every crate they like is "ergonomic". Whenever you cite (c), you must in the same breath name the `std` alternative, describe specifically what it lacks, and confirm the gap is large enough to outweigh the long-term-support cost. If you can't articulate that in one sentence, the case for (c) doesn't hold; fall back to `std`.

## Versioning and source-of-truth (second top-level rule)

For any third-party crate that **does** clear the std-first gate, two policies override the usual "pin a stable minor version" advice:

1. **Prefer the latest published version**, including pre-releases (`0.x`, `-rc`, `-beta`). This user is already on nightly Rust and accepts API churn. Don't pin to an old major just because "the API stabilized there".

2. **Prefer the GitHub `master`/`main` source over the crates.io release** when there is a meaningful gap — newer feature, bug fix not yet cut, or important API change merged but unreleased. Cargo supports this directly:

   ```toml
   [dependencies]
   # When crates.io release is sufficient:
   tokio = { version = "1", features = ["full"] }

   # When you need something newer than the latest release:
   tokio = { git = "https://github.com/tokio-rs/tokio", branch = "master", features = ["full"] }
   ```

   **Default to `branch = "master"` (or `"main"`).** The whole point of using a git dep is to track the moving frontier — `rev`-pinning defeats that.

   **Only switch to `rev = "<full-sha>"` when** the upstream crate has a track record of frequently breaking commonly-used public APIs and you've been burned (or expect to be). For most crates a branch dep is fine; `Cargo.lock` already records which commit was resolved, so reproducibility for a *given* build is preserved. `rev` is for protection against unexpected upstream churn on the next `cargo update`, not for routine use.

3. **Read the source when the docs lag.** For latest-version usage where docs.rs / README / examples are stale or absent, **go to the actual source on GitHub or `target/doc` and read it**. The `impl` is the spec; documentation is a hint. This is especially common for crates moving quickly (tokio, serde, jiff) or for unstable APIs hidden behind feature gates.

   Operational form:
   - Look at `src/lib.rs` / `src/<module>.rs` directly.
   - Look at the crate's own `tests/` and `examples/` directories — those are usually the freshest usage demos.
   - For ergonomic exploration, `cargo doc --open` produces local docs from the current source tree.

## Authoritative crate lists (live defaults — full rationale in references/crates-whitelist.md)

**Forbidden by default** (a `std`/stable replacement exists, use it):
`anyhow`, `thiserror`, `derive_more`, `parking_lot`, `ahash`, `smallvec`, `arrayvec`, `tinyvec`, `flume`, `async-trait`, `chrono`, `time` (the crate), `log`, `once_cell`, `lazy_static`, `itertools`, `walkdir`, `dirs`, `directories`.

**Approved** (no acceptable `std` equivalent — use freely):
`tokio`, `serde` + `serde_json` + `serde_derive`, `tracing` + `tracing-subscriber`, `criterion`, `proptest`, `insta`, `regex`, `clap` (derive), `rayon`, `bytemuck`, `bytes`, `num-traits`, `tempfile`, `jiff` (only when calendar/timezone needed; otherwise `std::time`), `cargo-nextest`, `cargo-llvm-cov`, `cargo-deny`, `cargo-machete`, `cargo-fuzz`, `bindgen`, `cbindgen`, `mimalloc` (only after benchmark).

When the user asks "should I use X?", check the lists first. If X is forbidden, name the `std` replacement explicitly and apply it. If X is on neither list, evaluate against the three-criteria gate and explain your reasoning.

## When to invoke

Invoke when **any** of these:

- Writing or editing `.rs` files in this user's projects.
- Designing a new Rust crate / workspace / module.
- Reviewing the user's Rust code or PRs.
- Answering "should I use X crate / should I use feature Y" Rust questions.
- Choosing between two Rust approaches (sync vs async, panic vs Result, generic vs `dyn`, etc.).
- Adding `Cargo.toml` dependencies.

Skip when:

- Pure language-theory questions ("what does the orphan rule mean", "explain Pin") with no code authoring involved.
- The user explicitly says "stable only" or "I'm constrained to no nightly" — in that case fall back to stable equivalents.
- The user is reviewing code from a project that has its own conventions (e.g. a public OSS contribution to a stable-only crate) — defer to that project's rules.

## Output behavior — what answers look like

When you propose Rust code, **explicitly call out** decisions tied to a skill rule the first time they appear in the file:

```rust
// nightly: generic_const_exprs
#![feature(generic_const_exprs)]
#![allow(incomplete_features)]
```

When you choose a `std` form over a "popular" third-party form, **note the trade-off in one line** so the user can override:

> Using `std::sync::Mutex` here (parking_lot would be marginally faster under heavy contention; benchmark before switching).

When the user explicitly requests a forbidden crate, **push back once with the std alternative**, apply it, and only fall back to the forbidden crate if the user re-affirms. Don't quietly comply.

## Project bootstrap (first contact with a Rust project)

When you first read a Rust project for this user and any of these files are missing at the project root, **copy them from this skill's `assets/`**:

| Missing file | Action |
|---|---|
| `rustfmt.toml` | Copy `assets/rustfmt.toml` to the project root |

Do this once, at the start of work on a new repo. Mention it to the user in one line ("seeding rustfmt.toml from your standard config — let me know if you want a different style for this repo"). Don't ask permission; the user has pre-authorized this.

The canonical `rustfmt.toml` shipped in `assets/` is the user's preferred config. It is more complete than the snippet in `style-lints-workspace.md` and supersedes that snippet — treat the asset file as the source of truth.

## Pre-commit workflow (Rust projects)

Whenever you're about to create a `git commit` in a Rust project (yours or the user's on their behalf), run this **before** staging or committing:

```bash
# 1. Apply clippy auto-fixes
cargo clippy --fix --allow-dirty --allow-staged --all-targets --all-features

# 2. Format
cargo fmt --all

# 3. Then stage + commit as usual
git add <specific paths>
git commit -m "..."
```

`--allow-dirty --allow-staged` is required because the working tree has the very changes you're about to commit; clippy refuses to auto-fix otherwise. `--all-targets --all-features` ensures tests/benches/examples are linted too.

If `cargo clippy --fix` modifies more than just formatting (rewrites expressions, changes types), **review the diff before staging**. The user wants the auto-fix applied but visible.

If either command fails, **stop**:
- `cargo clippy` failure → there are lints that can't be auto-fixed; surface them and fix manually before continuing.
- `cargo fmt` failure → almost always a syntax error somewhere; fix and retry.

Don't `--no-verify` past either. Skip this workflow only when:
- The commit is in a non-Rust subdirectory of a polyglot repo (no `Cargo.toml` in scope).
- The commit is a pure docs/markdown change with no `.rs` file modified.

## Reference files

Load when relevant — don't preload everything.

- `references/crates-whitelist.md` — Authoritative per-crate rationale: why each forbidden crate is forbidden, what to use instead, what each approved crate is for. **Open whenever a crate decision needs justification.**
- `references/toolchain.md` — `rust-toolchain.toml`, edition, the unstable-feature white/grey/black list.
- `references/error-handling.md` — Hand-written `impl std::error::Error` template, `Box<dyn Error + Send + Sync>` patterns, `std::backtrace::Backtrace` integration.
- `references/async.md` — `tokio` runtime conventions, async fn in trait, `spawn_blocking`, sync primitives inside async.
- `references/unsafe.md` — `SAFETY:` comment template, Miri policy, when unsafe is justified.
- `references/performance.md` — Allocator policy, `std::simd`, container selection, when to reach for `bytemuck` / `bytes` / `rayon` / `num-traits`.
- `references/testing.md` — `cargo nextest`, `insta`, `proptest`, `criterion`, doc-tests, Miri.
- `references/style-lints-workspace.md` — `rustfmt.toml`, clippy config + `disallowed_types`, workspace layout, `Cargo.toml` profile settings, pre-commit workflow details.
- `assets/rustfmt.toml` — the canonical rustfmt config to seed into projects on first contact (per "Project bootstrap" above). Treat as authoritative.
- `references/const-generics.md` — Compile-time parameterization patterns (struct-level, method-level, `generic_const_exprs`, `const fn`, enum dispatch, heap-allocated arrays, defaults + `PhantomData` anchoring).
- `references/const-generics-patterns-detail.md` — Extended const-generics reference: feature gate setup, stable vs nightly, where-bound mechanics, SIMD-friendly patterns, common pitfalls, dynamic→const-generic migration.

## What this skill is NOT

- Not a Rust tutorial. It assumes intermediate-to-advanced Rust fluency.
- Not a lint-rule list (rustfmt/clippy settings live in `style-lints-workspace.md`, but the skill doesn't enforce them tool-by-tool — it expresses *why* they're set that way).
- Not a substitute for benchmarking. Whenever a rule says "prefer std" or "prefer X over Y", that's the default — measurement always trumps the rule.
