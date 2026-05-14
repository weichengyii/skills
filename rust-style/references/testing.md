# Testing

## Runner: `cargo-nextest`

Default test runner. Faster, better output, real-time progress.

```bash
cargo nextest run
cargo nextest run --workspace
cargo nextest run -p my-crate test_name
```

CI: install `cargo-nextest` and use it. Don't use `cargo test` in CI — its output truncates and parallelism is worse.

## Snapshot tests: `insta` (whitelisted)

For assertions where the expected output is complex / multi-line / matches a Display / matches generated code:

```rust
#[test]
fn renders_header() {
    let out = render(&example_input());
    insta::assert_snapshot!(out);
}
```

First run creates `tests/snapshots/<test>.snap`. Subsequent runs diff against it. Review with `cargo insta review`.

When NOT to use: simple equality (`assert_eq!`), single-field checks (`assert!(x.is_empty())`). Snapshot tests are heavyweight for trivial cases.

## Property tests: `proptest` (whitelisted)

For invariants ("for all inputs X, P(X) holds") and round-trips:

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn parse_then_render_is_identity(s: String) {
        let parsed = MyType::parse(&s).expect("parse");
        let rendered = parsed.to_string();
        let reparsed = MyType::parse(&rendered).expect("reparse");
        prop_assert_eq!(parsed, reparsed);
    }
}
```

Default: 256 cases. For expensive properties, lower via `#![proptest_config(ProptestConfig::with_cases(64))]`.

## Benchmarks: `criterion` (whitelisted)

Place in `benches/`. Use Criterion for **all** benchmarks; never the unstable `#[bench]` attribute.

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_parse(c: &mut Criterion) {
    let input = include_str!("../tests/fixtures/large.txt");
    c.bench_function("parse_large", |b| {
        b.iter(|| my_crate::parse(black_box(input)));
    });
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

Run: `cargo bench`. CI: gate on regressions with `cargo bench -- --baseline main`.

## Fuzz tests: `cargo-fuzz`

For parsers, deserializers, and anything that takes attacker-influenced bytes:

```bash
cargo fuzz init
cargo fuzz run my_parser
```

Run in CI on a schedule (not every commit) — fuzzing is slow.

## Miri: required for unsafe code

If a crate uses `unsafe`, CI must run:

```bash
cargo +nightly miri test
```

Add a `miri` job in `.github/workflows/ci.yml`. Failing miri = soundness bug; treat as a release blocker.

## Coverage: `cargo-llvm-cov`

```bash
cargo llvm-cov --workspace --html
```

Targets:

- Aim for ≥80% line coverage on library crates.
- Don't chase 100% — diminishing returns past 85%. Error-handling branches that "can't happen" don't need synthetic coverage.

## Doc tests

Every public item gets a doc comment with a `# Examples` section containing a runnable example. CI runs `cargo test --doc` to verify they compile.

```rust
/// Compute the dot product of two slices.
///
/// # Examples
///
/// ```
/// use my_crate::dot;
/// assert_eq!(dot(&[1.0, 2.0], &[3.0, 4.0]), 11.0);
/// ```
pub fn dot(a: &[f32], b: &[f32]) -> f32 { /* ... */ }
```

Doc tests run as integration tests — they catch API breakages that unit tests would miss.

## Test organization

- **Unit tests** — `#[cfg(test)] mod tests { ... }` inside the file under test. For private-function tests and tight feedback loops.
- **Integration tests** — `tests/<name>.rs` at crate root. For testing the public API.
- **Doc tests** — required for public items.
- **Fixtures** — `tests/fixtures/` with sample data, loaded via `include_str!` / `include_bytes!`.

## What NOT to use

- `mockall` / `mockito` / mock crates — prefer trait-based test doubles you hand-write (see error-handling for the no-derive-macro reasoning). 
- `serial_test` — if tests need ordering, restructure them (or use a `Mutex` in the test module).
- `pretty_assertions` — `assert_eq!` is sufficient. The extra diff output is nice but doesn't justify a dependency on every test build.
