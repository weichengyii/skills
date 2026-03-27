---
name: rust-const-generics
description: Guide for applying Rust const generics patterns to achieve zero-overhead, type-safe compile-time parameterization. Use this skill whenever the user is designing Rust structs/traits/functions where dimensions, sizes, or capacities could be compile-time constants — especially for numeric computing, data pipelines, protocol buffers, neural networks, DSP, fixed-size containers, or any domain where array sizes and structural parameters should be statically verified. Also trigger when the user mentions "const generics", "generic_const_exprs", compile-time array sizing, or wants to eliminate runtime bounds checks through type-level programming.
---

# Rust Const Generics Patterns

This skill teaches you how to use Rust's const generics system to encode structural parameters (dimensions, sizes, capacities) as compile-time constants. The result: zero-overhead abstractions where the compiler verifies all size relationships and can fully optimize loops and memory layout.

## When to Apply Const Generics

Const generics shine when your types have **structural parameters** — values that define shape rather than behavior:

- Array/buffer dimensions (channel count, sample count, vector length)
- Protocol field sizes (packet length, header width)
- Algorithm parameters (kernel size, stride, block size)
- Container capacities (ring buffer depth, pool size)

The key question: **"Would getting this size wrong at runtime cause a logic bug?"** If yes, encode it as a const generic so the compiler catches mismatches.

## Pattern Catalog

### Pattern 1: Struct-Level Shape Parameters

Encode your type's fundamental dimensions as const generics on the struct itself. These are values that are fixed for the lifetime of the instance.

```rust
pub struct RingBuffer<const CAPACITY: usize> {
    data: [f32; CAPACITY],
    head: usize,
}

pub struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f32; COLS]; ROWS],
}

// Multi-dimensional: encode every axis that defines the type's shape
pub struct ConvKernel<const IN_CH: usize, const OUT_CH: usize, const SIZE: usize> {
    weights: [[[f32; SIZE]; IN_CH]; OUT_CH],
    bias: [f32; OUT_CH],
}
```

**When to use:** The parameter is intrinsic to the type and doesn't change between method calls.

### Pattern 2: Method-Level Parameters

Some dimensions vary per call rather than per instance. Add these as const generics on the method.

```rust
impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    /// BATCH_SIZE varies per call, not per Matrix instance
    pub fn transform_batch<const BATCH_SIZE: usize>(
        &self,
        input: &[[f32; COLS]; BATCH_SIZE],
        output: &mut [[f32; ROWS]; BATCH_SIZE],
    ) {
        // Compiler knows exact loop bounds — can unroll and vectorize
        for b in 0..BATCH_SIZE {
            for r in 0..ROWS {
                let mut sum = 0.0;
                for c in 0..COLS {
                    sum += self.data[r][c] * input[b][c];
                }
                output[b][r] = sum;
            }
        }
    }
}
```

**When to use:** The parameter affects the I/O shape of a single operation but isn't part of the type's identity.

### Pattern 3: Const Expression Where Bounds (`generic_const_exprs`)

When you need arithmetic on const generics (e.g., derived buffer sizes), use the `[(); EXPR]:` where-bound pattern. This requires nightly and the `generic_const_exprs` feature.

```rust
#![feature(generic_const_exprs)]
#![allow(incomplete_features)]

pub struct Pipeline<const IN: usize, const STRIDE: usize>
where
    [(); IN / STRIDE]:,        // proves IN is divisible by STRIDE
    [(); (IN - 1) * STRIDE]:,  // proves the expression is a valid array length
{
    input_buf: [f32; IN],
    downsampled_buf: [f32; IN / STRIDE],  // derived size
    state: [f32; (IN - 1) * STRIDE],      // another derived size
}
```

The `[(); EXPR]:` idiom forces the compiler to evaluate the const expression and verify it produces a valid `usize`. Without it, the compiler rejects arithmetic on const generics.

**Supported expressions:** `+`, `-`, `*`, `/`, `%`, `<<`, `>>`, and calls to `const fn`.

**When to use:** Whenever a field's size is computed from the struct's const generic parameters rather than being a parameter itself.

### Pattern 4: `const fn` as Type-Level Computation

For complex size calculations, define `const fn` helpers and use them in generic positions. This keeps where-clauses manageable.

```rust
/// Compute channel count at a given level of a hierarchical architecture
pub const fn ch(base: usize, level: usize, max: usize) -> usize {
    let raw = base << level;  // double channels each level
    if raw < max { raw } else { max }
}

/// Compute sample count at a given level
pub const fn samples(chunk: usize, stride: usize, level: usize) -> usize {
    chunk / stride.pow(level as u32)
}

// Use in type positions with {} braces:
pub struct UNet<const BASE_CH: usize, const MAX_CH: usize, const STRIDE: usize, const CHUNK: usize>
where
    [(); ch(BASE_CH, 0, MAX_CH)]:,
    [(); ch(BASE_CH, 1, MAX_CH)]:,
    [(); samples(CHUNK, STRIDE, 1)]:,
{
    level0: Layer<{ ch(BASE_CH, 0, MAX_CH) }, { ch(BASE_CH, 1, MAX_CH) }>,
    buf_l1: [f32; samples(CHUNK, STRIDE, 1)],
}
```

**When to use:** When you have a family of related sizes that follow a formula (e.g., hierarchical architectures, multi-resolution pipelines).

#### Combining `const fn` with arithmetic in where bounds

`const fn` calls can freely compose with arithmetic operators inside `[(); EXPR]:` bounds. This is how real-world hierarchical architectures verify all derived sizes at compile time:

```rust
pub struct Codec<
    const BASE_CH: usize,
    const MAX_CH: usize,
    const STRIDE: usize,
    const CHUNK: usize,
>
where
    // const fn result + arithmetic: prove half-channel splits are valid
    [(); ch(BASE_CH, 0, MAX_CH) / 2]:,
    [(); ch(BASE_CH, 1, MAX_CH) / 2]:,

    // const fn result + multiplication: prove FiLM double-width weights are valid
    [(); ch(BASE_CH, 0, MAX_CH) * 2]:,
    [(); ch(BASE_CH, 1, MAX_CH) * 2]:,

    // two const fn calls combined: prove skip-connection concat size is valid
    [(); ch(BASE_CH, 0, MAX_CH) + ch(BASE_CH, 0, MAX_CH)]:,

    // two different const fns multiplied: prove per-level buffer total size
    [(); ch(BASE_CH, 0, MAX_CH) * samples(CHUNK, STRIDE, 0)]:,
    [(); ch(BASE_CH, 1, MAX_CH) * samples(CHUNK, STRIDE, 1)]:,

    // direct arithmetic on params: prove scratch buffer sizes
    [(); MAX_CH * CHUNK]:,
    [(); 2 * MAX_CH * CHUNK]:,
{
    // Fields use the same expressions — all proven valid by the where bounds above
    enc0: EncoderBlock<{ ch(BASE_CH, 0, MAX_CH) }, { ch(BASE_CH, 1, MAX_CH) }, STRIDE>,
    skip_buf_0: Box<[f32; ch(BASE_CH, 0, MAX_CH) * samples(CHUNK, STRIDE, 0)]>,
    scratch: Box<[f32; MAX_CH * CHUNK]>,
}
```

The rule: **any expression that appears as an array length in a field type must also appear in a `[(); EXPR]:` where bound.** The expression can be arbitrarily complex — nested `const fn` calls, mixed with `+`, `*`, `/` — as long as it evaluates to a valid `usize` at monomorphization time.

### Pattern 5: Associated Constants from Generic Params

Expose derived values as associated constants so callers can query them without knowing the formula.

```rust
impl<const KERNEL: usize, const DILATION: usize> CausalConv<KERNEL, DILATION>
where
    [(); (KERNEL - 1) * DILATION]:,
{
    /// Receptive field minus one — useful for computing required context
    pub const RECEPTIVE_FIELD_M1: usize = (KERNEL - 1) * DILATION;

    /// Total receptive field
    pub const RECEPTIVE_FIELD: usize = Self::RECEPTIVE_FIELD_M1 + 1;
}
```

**When to use:** When callers need to know a derived dimension (e.g., to allocate buffers, chain components).

### Pattern 6: Enum Dispatch for Heterogeneous Collections

When you need a `Vec` of items that differ only in one const generic value, use an enum with one variant per concrete value. This preserves static dispatch while allowing runtime iteration.

```rust
pub enum BlockVariant<const CH: usize> {
    D1(Block<CH, 1>),
    D2(Block<CH, 2>),
    D4(Block<CH, 4>),
    D8(Block<CH, 8>),
}

impl<const CH: usize> BlockVariant<CH> {
    pub fn process<const SAMPLES: usize>(&mut self, data: &mut [[f32; SAMPLES]; CH]) {
        match self {
            Self::D1(b) => b.process(data),
            Self::D2(b) => b.process(data),
            Self::D4(b) => b.process(data),
            Self::D8(b) => b.process(data),
        }
    }
}

// Now you can have: Vec<BlockVariant<64>>
```

**When to use:** When you need a collection of const-generic types that share most parameters but differ in one, and you want to avoid `dyn Trait` / vtable overhead. The set of variants must be known at compile time.

### Pattern 7: Heap Allocation for Large Const-Generic Arrays

Stack-allocated `[T; N]` where N is large will overflow the stack. Use `Box` for heap placement while keeping the compile-time size guarantee.

```rust
pub struct LargeModel<const CH: usize, const SAMPLES: usize> {
    buf: Box<[[f32; SAMPLES]; CH]>,  // heap-allocated, but size is still compile-time known
}

impl<const CH: usize, const SAMPLES: usize> LargeModel<CH, SAMPLES> {
    pub fn new() -> Self {
        Self {
            buf: Box::new([[0.0; SAMPLES]; CH]),
        }
    }
}
```

**When to use:** When `CH * SAMPLES * size_of::<f32>()` exceeds a few hundred KB.

## Decision Guide

```
Is the parameter fixed for the type's lifetime?
├── Yes → Struct-level const generic (Pattern 1)
│   └── Is any field size derived from this param?
│       ├── Simple expression → [(); EXPR]: where bound (Pattern 3)
│       └── Complex formula → const fn helper (Pattern 4)
└── No, varies per call → Method-level const generic (Pattern 2)

Need a Vec of differently-parameterized items?
├── Known set of values → Enum dispatch (Pattern 6)
└── Open-ended → Consider dyn Trait instead

Array too large for stack?
└── Yes → Box<[T; N]> (Pattern 7)
```

## Checklist Before Using Const Generics

1. **Do you actually need compile-time sizes?** If the size is truly dynamic (user input, file-dependent), use `Vec`. Const generics are for structurally fixed dimensions.
2. **Nightly required?** Basic `<const N: usize>` is stable. Arithmetic on const generics (`generic_const_exprs`) requires nightly.
3. **Monomorphization cost acceptable?** Each unique combination of const values generates a new copy of the code. For 2-3 parameters with small value ranges, this is fine. For combinatorial explosions (many params, many values), consider whether the performance gain justifies the binary size.
4. **Are where-bounds getting unwieldy?** If you have more than ~5 where bounds, extract size computation into `const fn` helpers (Pattern 4) and/or add intermediate type aliases.

## Detailed Reference

For extended examples, migration strategies, and pitfalls, see `references/patterns-detail.md`.
