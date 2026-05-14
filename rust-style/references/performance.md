# Performance

## Allocator

**Default: the system allocator.** Don't switch unless a profiler shows allocation in the top of CPU samples. When you do switch, `mimalloc` is the first stop:

```toml
[dependencies]
mimalloc = "0.1"
```

```rust
#[global_allocator]
static GLOBAL: mimalloc::MiMalloc = mimalloc::MiMalloc;
```

Document the benchmark that justified it.

## Containers — choose the std type that fits the use case

| Use case | Type |
|---|---|
| Heap-allocated growable sequence | `Vec<T>` |
| Compile-time fixed size, stack | `[T; N]` (const generic where dimension matters) |
| Compile-time fixed size, heap (large) | `Box<[T; N]>` |
| Key/value map, no ordering | `std::collections::HashMap<K, V>` |
| Key/value map, ordered iteration | `std::collections::BTreeMap<K, V>` |
| Set | `HashSet` / `BTreeSet` |
| Double-ended queue | `VecDeque<T>` |
| Linked list | Almost never. Use `Vec` + indices for graph-like structures |

**Do not reach for `smallvec`, `arrayvec`, `tinyvec`, `indexmap` reflexively.** Profile first. For stack-allocated small collections, `[T; N]` or `[MaybeUninit<T>; N]` + `usize len` (~30 lines) is preferred.

## SIMD — `std::simd` (portable_simd)

Default SIMD entry point is `std::simd` (portable_simd nightly feature). Cleanly expresses width-generic operations:

```rust
#![feature(portable_simd)]
use std::simd::{f32x8, Simd, SimdFloat};

pub fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    let chunks = a.chunks_exact(8).zip(b.chunks_exact(8));
    let mut acc = f32x8::splat(0.0);
    for (a_chunk, b_chunk) in chunks {
        let va = f32x8::from_slice(a_chunk);
        let vb = f32x8::from_slice(b_chunk);
        acc += va * vb;
    }
    acc.reduce_sum()
        + a.chunks_exact(8).remainder().iter()
            .zip(b.chunks_exact(8).remainder())
            .map(|(x, y)| x * y)
            .sum::<f32>()
}
```

Reach for `core::arch::x86_64::*` intrinsics only when portable_simd can't express the operation (e.g. AVX-512 mask ops, AES-NI, CRC32 specifics).

Never use the `wide` crate or third-party SIMD wrappers — `std::simd` is the std-first choice.

## Parallelism — `rayon` for data, std for control

- **Data parallel** (map / filter / reduce over a collection): `rayon::prelude::*` and `.par_iter()`. Whitelisted under criterion (c).
- **Control parallel** (a few specific tasks): `std::thread::scope`. Don't pull in rayon for 3 long-running threads.

```rust
use std::thread;

thread::scope(|s| {
    s.spawn(|| do_work_a());
    s.spawn(|| do_work_b());
    s.spawn(|| do_work_c());
});  // joins automatically
```

Mixed: inside async, use `tokio::task::spawn_blocking` then `rayon` inside that block (see `async.md`).

## Zero-copy / byte-level

- `bytemuck::cast_slice`, `bytemuck::Pod`, `bytemuck::Zeroable` — safe alternative to `transmute` for byte-level work. Whitelisted.
- `bytes::Bytes` / `bytes::BytesMut` — networking and streaming zero-copy. Whitelisted.

Don't write `unsafe { std::mem::transmute }` when bytemuck can express it safely.

## Generic numeric code — `num-traits`

Whitelisted for writing algorithms that work over `f32` / `f64` / `i32` / `i64` generically. Don't hand-roll your own `Float` / `Integer` traits.

```rust
use num_traits::Float;

pub fn rms<F: Float>(samples: &[F]) -> F {
    let sum_sq = samples.iter().map(|x| x.powi(2)).fold(F::zero(), |a, b| a + b);
    (sum_sq / F::from(samples.len()).unwrap()).sqrt()
}
```

## Hashing

Stick with the default `std` `HashMap` (SipHash, HashDoS-safe). Only switch hashers when:

- The keys are not attacker-influenced.
- A profiler shows hashing is in the top 5 of CPU.
- You can document the benchmark in code comments.

Then `ahash::AHashMap` is fine — but be explicit, don't apply it globally.

## Cache-friendliness checklist

When perf matters:

1. **Struct-of-arrays vs array-of-structs** — prefer SoA for hot loops (better SIMD, better cache).
2. **Avoid `Vec<Box<T>>`** unless `T` is large or polymorphic — `Vec<T>` for owned values keeps memory contiguous.
3. **Align hot data to cache lines** with `#[repr(align(64))]` when sharing across threads.
4. **Profile-guided optimization (PGO)** — for release builds where the workload is stable.

## When profiling

Default tools:

- `cargo flamegraph` (via `cargo-flamegraph`) — CPU flame graphs.
- `perf` directly on Linux for accurate samples.
- `criterion` for micro-benchmarks (whitelisted).
- Don't add a profiling dependency at runtime — instrumented builds skew results.
