# Const Generics Detailed Reference

## Table of Contents
1. [Feature Gate Setup](#feature-gate-setup)
2. [Stable vs Nightly Features](#stable-vs-nightly)
3. [The Where-Bound Trick Explained](#where-bound-trick)
4. [Composing Const Generics Across Types](#composing)
5. [SIMD-Friendly Patterns](#simd)
6. [Common Pitfalls](#pitfalls) (including unconstrained const generics + PhantomData fix)
7. [Migration: From Dynamic to Const Generic](#migration)

---

## Feature Gate Setup <a name="feature-gate-setup"></a>

For basic const generics (stable Rust):
```rust
// No feature gate needed — works on stable since Rust 1.51
struct Buffer<const N: usize> {
    data: [u8; N],
}
```

For const expressions in generic positions (nightly):
```rust
#![feature(generic_const_exprs)]
#![allow(incomplete_features)]  // suppress the "incomplete" warning
```

Place these at the crate root (`lib.rs` or `main.rs`). They apply crate-wide.

---

## Stable vs Nightly Features <a name="stable-vs-nightly"></a>

| Feature | Stability | Example |
|---------|-----------|---------|
| `<const N: usize>` on structs/functions | **Stable** (1.51+) | `struct Buf<const N: usize>` |
| `[T; N]` where N is a const param | **Stable** | `data: [f32; N]` |
| Arithmetic in type positions | **Nightly** | `buf: [f32; N * 2]` |
| `const fn` results in type positions | **Nightly** | `buf: [f32; compute(N)]` |
| `[(); EXPR]:` where bounds | **Nightly** | `where [(); N / 2]:` |
| Const generic defaults | **Nightly** | `<const N: usize = 64>` |
| Dependent const generic defaults | **Nightly** | `<const N: usize, const D: usize = { N / 2 }>` |
| `f32` arithmetic in `const fn` | **Nightly** | `const fn f(x: f32) -> usize { x as usize }` |
| `f32` as const generic param type | **Not supported** | `<const S: f32>` ❌ (NaN equality) |

**Recommendation:** If you only need simple `<const N: usize>` without arithmetic, stay on stable. If you need derived sizes, accept nightly and use `generic_const_exprs`.

### Important distinctions

**`f32` in const fn vs as const generic type** — these are different things:
- `const fn foo(scale: f32) -> usize` — ✅ works on nightly, `f32` is a runtime parameter to the const fn
- `<const SCALE: f32>` — ❌ not allowed, `f32` doesn't implement `ConstParamTy` due to NaN
- Workaround: pass `f32` into `const fn`, return `usize`, use the `usize` result as a const generic default

---

## The Where-Bound Trick Explained <a name="where-bound-trick"></a>

Why does `[(); (K - 1) * D]:` work as a constraint?

1. `[(); EXPR]` is an array of `EXPR` zero-sized unit types
2. For this type to exist, `EXPR` must be a valid `usize` (non-negative, no overflow)
3. By adding it to a `where` clause, we tell the compiler: "this expression is guaranteed to be valid wherever this type is used"
4. The compiler then allows `EXPR` to appear in field types

Without the where bound:
```rust
// ERROR: unconstrained generic constant
struct Foo<const N: usize> {
    buf: [f32; N * 2],  // compiler can't prove N * 2 is valid
}
```

With the where bound:
```rust
// OK: the where bound proves N * 2 is a valid array length
struct Foo<const N: usize> where [(); N * 2]: {
    buf: [f32; N * 2],
}
```

**Every distinct arithmetic expression needs its own where bound.** If you use both `N * 2` and `N / 2`, you need:
```rust
where
    [(); N * 2]:,
    [(); N / 2]:,
```

### Propagation rule

Where bounds must appear on every impl block and function that uses the expression, not just the struct definition:

```rust
struct Foo<const N: usize> where [(); N * 2]: {
    buf: [f32; N * 2],
}

// The impl MUST repeat the where bound
impl<const N: usize> Foo<N> where [(); N * 2]: {
    fn new() -> Self {
        Self { buf: [0.0; N * 2] }
    }
}

// Any function that constructs or uses Foo must also carry the bound
fn make_foo<const N: usize>() -> Foo<N> where [(); N * 2]: {
    Foo::new()
}
```

### Eliminating where-bound repetition with const generic defaults

If the repeated where bounds become burdensome, refactor to const generic defaults. The derived sizes become independent const generic parameters with default values, so no where bounds are needed:

```rust
// Before: where bounds repeated on struct + every impl
struct Foo<const N: usize> where [(); N * 2]:, [(); N / 4]: {
    a: [f32; N * 2],
    b: [f32; N / 4],
}
impl<const N: usize> Foo<N> where [(); N * 2]:, [(); N / 4]: { ... }

// After: defaults compute the sizes, no where bounds anywhere
struct Foo<
    const N: usize,
    const A: usize = { N * 2 },
    const B: usize = { N / 4 },
> {
    _phantom: core::marker::PhantomData<[(); N]>,
    a: [f32; A],
    b: [f32; B],
}
impl<const N: usize, const A: usize, const B: usize> Foo<N, A, B> { ... }
```

This trades type-parameter verbosity for zero where-bound repetition. Users still write `Foo::<64>` and get `A=128, B=16` automatically, or override with `Foo::<64, 256, 32>`.

---

## Composing Const Generics Across Types <a name="composing"></a>

The real power of const generics emerges when types compose — parameters flow from outer types to inner types:

```rust
struct Inner<const CH: usize> {
    weights: [f32; CH],
}

struct Middle<const CH: usize, const LATENT: usize>
where [(); CH * 2]:
{
    inner: Inner<CH>,
    film_weights: [[f32; CH * 2]; LATENT],
}

struct Outer<const CH: usize, const LATENT: usize, const SAMPLES: usize>
where
    [(); CH * 2]:,
    [(); SAMPLES / 2]:,
{
    stage: Middle<CH, LATENT>,           // CH and LATENT flow down
    buf: Box<[[f32; SAMPLES]; CH]>,      // SAMPLES used at this level
    downsampled: Box<[[f32; SAMPLES / 2]; CH]>,  // derived size
}

// Instantiate with concrete values:
type MyModel = Outer<64, 32, 48>;
```

**Type aliases** are useful for concrete instantiations:
```rust
pub type StandardDecoder = Outer<64, 32, 48>;
pub type LargeDecoder = Outer<128, 64, 96>;
```

---

## SIMD-Friendly Patterns <a name="simd"></a>

Const generic loop bounds enable the compiler to generate optimal SIMD code:

```rust
#![feature(portable_simd)]
use std::simd::f32x8;

fn dot_product<const N: usize>(a: &[f32; N], b: &[f32; N]) -> f32 {
    let mut sum = f32x8::splat(0.0);
    let mut i = 0;

    // Compiler knows exactly how many SIMD iterations:
    // N / 8 full iterations + possible scalar tail
    while i + 8 <= N {
        let va = f32x8::from_slice(&a[i..]);
        let vb = f32x8::from_slice(&b[i..]);
        sum += va * vb;
        i += 8;
    }

    let mut result = sum.reduce_sum();

    // Scalar tail for remaining elements
    while i < N {
        result += a[i] * b[i];
        i += 1;
    }

    result
}
```

Because `N` is a compile-time constant, the compiler can:
- Unroll the SIMD loop completely for small N
- Eliminate the scalar tail if N is a multiple of 8
- Remove all bounds checks on array accesses

---

## Common Pitfalls <a name="pitfalls"></a>

### 1. Forgetting where bounds on impl blocks

```rust
struct Foo<const N: usize> where [(); N * 2]: { /* ... */ }

// WRONG: missing where bound
impl<const N: usize> Foo<N> {
    fn new() -> Self { /* ... */ }  // won't compile
}

// RIGHT: repeat the bound
impl<const N: usize> Foo<N> where [(); N * 2]: {
    fn new() -> Self { /* ... */ }
}
```

### 2. Stack overflow from large arrays

```rust
// DANGEROUS if CH * SAMPLES is large:
struct Model<const CH: usize, const SAMPLES: usize> {
    buf: [[f32; SAMPLES]; CH],  // lives on the stack!
}

// SAFE: heap-allocate
struct Model<const CH: usize, const SAMPLES: usize> {
    buf: Box<[[f32; SAMPLES]; CH]>,
}
```

### 3. Monomorphization explosion

Each unique combination generates separate machine code:
```rust
// This creates 100 * 100 = 10,000 monomorphized copies!
for rows in 1..=100 {
    for cols in 1..=100 {
        // Matrix<rows, cols>::new()  — can't do this anyway, but illustrates the concept
    }
}
```

In practice, you typically instantiate with a small, fixed set of values (e.g., one model configuration = one monomorphization).

### 4. Trait implementations need bounds too

```rust
// Display also needs the where bound
impl<const N: usize> std::fmt::Display for Foo<N>
where [(); N * 2]:
{
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "Foo with {} elements", N * 2)
    }
}
```

### 5. Unconstrained const generic from default-value-only usage

When a const generic is used only in default-value expressions for other const generics (not in any field type), the compiler reports `unconstrained generic constant` on impl blocks:

```rust
// ❌ MAX_S only appears in D0's default — unconstrained in impl
pub struct Foo<const MAX_S: usize, const D0: usize = { MAX_S / 2 }> {
    buf: [f32; D0],
}

impl<const MAX_S: usize, const D0: usize> Foo<MAX_S, D0> {
    fn new() -> Self { ... }  // ERROR: unconstrained generic constant `MAX_S`
}
```

The fix: anchor `MAX_S` in a field type via `PhantomData`:

```rust
// ✅ PhantomData makes MAX_S "used" in the type — zero runtime cost
pub struct Foo<const MAX_S: usize, const D0: usize = { MAX_S / 2 }> {
    _phantom: core::marker::PhantomData<[(); MAX_S]>,
    buf: [f32; D0],
}

impl<const MAX_S: usize, const D0: usize> Foo<MAX_S, D0> {
    fn new() -> Self {
        Self { _phantom: core::marker::PhantomData, buf: [0.0; D0] }
    }
}
```

**Do not** use `[(); MAX_S]` directly as a field — that allocates `MAX_S` bytes on the stack. `PhantomData<[(); MAX_S]>` is zero-sized.

### 6. Division by zero in const expressions

```rust
// This compiles but panics if STRIDE == 0 at monomorphization time
struct Foo<const N: usize, const STRIDE: usize>
where [(); N / STRIDE]:
{
    buf: [f32; N / STRIDE],
}

// Foo<64, 0> will fail to compile — but the error message may be cryptic
```

Add a `debug_assert!` or document that STRIDE must be nonzero.

---

## Migration: From Dynamic to Const Generic <a name="migration"></a>

### Before (dynamic sizes)
```rust
struct Processor {
    channels: usize,
    buf: Vec<Vec<f32>>,
}

impl Processor {
    fn new(channels: usize, samples: usize) -> Self {
        Self {
            channels,
            buf: vec![vec![0.0; samples]; channels],
        }
    }

    fn process(&mut self, input: &[Vec<f32>], output: &mut [Vec<f32>]) {
        assert_eq!(input.len(), self.channels);  // runtime check
        // ... bounds checks on every access
    }
}
```

### After (const generic)
```rust
struct Processor<const CH: usize> {
    buf: Box<[Vec<f32>; CH]>,  // or Box<[[f32; SAMPLES]; CH]> if SAMPLES is also const
}

impl<const CH: usize> Processor<CH> {
    fn process<const SAMPLES: usize>(
        &mut self,
        input: &[[f32; SAMPLES]; CH],    // shape enforced by type system
        output: &mut [[f32; SAMPLES]; CH],
    ) {
        // No assert needed — wrong shapes won't compile
        for ch in 0..CH {
            for s in 0..SAMPLES {
                output[ch][s] = input[ch][s] * 2.0;  // no bounds check
            }
        }
    }
}
```

### Migration checklist
1. Identify which dimensions are structurally fixed (known at compile time)
2. Replace those `usize` fields with `const` generic parameters
3. Replace `Vec<T>` with `[T; N]` (or `Box<[T; N]>` for large arrays)
4. Replace runtime `assert_eq!` on sizes with compile-time type constraints
5. Add `[(); EXPR]:` where bounds for any derived sizes
6. Add `#![feature(generic_const_exprs)]` if using arithmetic on const params
7. Update all impl blocks and consuming functions to carry the where bounds
