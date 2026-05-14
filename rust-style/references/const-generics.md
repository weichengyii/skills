# Const generics — compile-time parameterization

Use const generics to encode structural parameters (dimensions, sizes, capacities) as compile-time constants. Result: zero-overhead abstractions where the compiler verifies all size relationships and can fully optimize loops and memory layout.

Open `const-generics-patterns-detail.md` for the deep dive (feature gate setup, stable vs nightly, where-bound mechanics, SIMD-friendly patterns, common pitfalls, migration from dynamic types).

## When to apply

Const generics shine when your types have **structural parameters** — values that define shape rather than behavior:

- Array/buffer dimensions (channel count, sample count, vector length)
- Protocol field sizes (packet length, header width)
- Algorithm parameters (kernel size, stride, block size)
- Container capacities (ring buffer depth, pool size)

The key question: **"Would getting this size wrong at runtime cause a logic bug?"** If yes, encode it as a const generic so the compiler catches mismatches.

## Pattern catalog

### Pattern 1 — Struct-level shape parameters

Encode the type's fundamental dimensions on the struct itself. Use when the parameter is fixed for the type's lifetime.

```rust
pub struct RingBuffer<const CAPACITY: usize> {
    data: [f32; CAPACITY],
    head: usize,
}

pub struct Matrix<const ROWS: usize, const COLS: usize> {
    data: [[f32; COLS]; ROWS],
}

pub struct ConvKernel<const IN_CH: usize, const OUT_CH: usize, const SIZE: usize> {
    weights: [[[f32; SIZE]; IN_CH]; OUT_CH],
    bias: [f32; OUT_CH],
}
```

### Pattern 2 — Method-level parameters

Some dimensions vary per call rather than per instance. Add these on the method.

```rust
impl<const ROWS: usize, const COLS: usize> Matrix<ROWS, COLS> {
    pub fn transform_batch<const BATCH_SIZE: usize>(
        &self,
        input: &[[f32; COLS]; BATCH_SIZE],
        output: &mut [[f32; ROWS]; BATCH_SIZE],
    ) {
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

### Pattern 3 — Const expressions in where bounds (`generic_const_exprs`)

For arithmetic on const generics, use the `[(); EXPR]:` where-bound pattern. Requires nightly + `generic_const_exprs`.

```rust
#![feature(generic_const_exprs)]
#![allow(incomplete_features)]

pub struct Pipeline<const IN: usize, const STRIDE: usize>
where
    [(); IN / STRIDE]:,
    [(); (IN - 1) * STRIDE]:,
{
    input_buf: [f32; IN],
    downsampled_buf: [f32; IN / STRIDE],
    state: [f32; (IN - 1) * STRIDE],
}
```

Supported expressions: `+`, `-`, `*`, `/`, `%`, `<<`, `>>`, and `const fn` calls.

### Pattern 4 — `const fn` as type-level computation

For complex size calculations, define `const fn` helpers and use them in generic positions. Nightly supports f32/f64 arithmetic inside `const fn`.

```rust
pub const fn ch(base: usize, level: usize, max: usize) -> usize {
    let raw = base << level;
    if raw < max { raw } else { max }
}

pub const fn delay_size(max_samples: usize, scale: f32) -> usize {
    (max_samples as f32 * scale) as usize
}

pub struct UNet<const BASE_CH: usize, const MAX_CH: usize, const STRIDE: usize, const CHUNK: usize>
where
    [(); ch(BASE_CH, 0, MAX_CH)]:,
    [(); ch(BASE_CH, 1, MAX_CH)]:,
{
    level0: Layer<{ ch(BASE_CH, 0, MAX_CH) }, { ch(BASE_CH, 1, MAX_CH) }>,
    buf_l1: [f32; ch(BASE_CH, 0, MAX_CH)],
}
```

`const fn` calls compose freely with arithmetic inside `[(); EXPR]:` bounds. Rule: **every expression that appears as an array length in a field must also appear in a `[(); EXPR]:` where bound.**

### Pattern 5 — Associated constants from generic params

Expose derived values as associated constants:

```rust
impl<const KERNEL: usize, const DILATION: usize> CausalConv<KERNEL, DILATION>
where
    [(); (KERNEL - 1) * DILATION]:,
{
    pub const RECEPTIVE_FIELD_M1: usize = (KERNEL - 1) * DILATION;
    pub const RECEPTIVE_FIELD: usize = Self::RECEPTIVE_FIELD_M1 + 1;
}
```

### Pattern 6 — Enum dispatch for heterogeneous collections

When you need a `Vec` of items that differ only in one const generic value, use an enum:

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
```

Preserves static dispatch while allowing runtime iteration. The set of variants must be known at compile time.

### Pattern 7 — Heap allocation for large arrays

Stack arrays where N is large will overflow the stack. Use `Box`:

```rust
pub struct LargeModel<const CH: usize, const SAMPLES: usize> {
    buf: Box<[[f32; SAMPLES]; CH]>,
}

impl<const CH: usize, const SAMPLES: usize> LargeModel<CH, SAMPLES> {
    pub fn new() -> Self {
        Self {
            buf: Box::new([[0.0; SAMPLES]; CH]),
        }
    }
}
```

Apply when `CH * SAMPLES * size_of::<T>()` exceeds a few hundred KB.

### Pattern 8 — Const generic defaults with dependent values

When derived parameters follow a standard ratio but should be overridable, use defaults that reference earlier parameters. This eliminates repeated where bounds.

```rust
pub const fn delay_size(max_samples: usize, scale: f32) -> usize {
    (max_samples as f32 * scale) as usize
}

pub struct Reverb<
    const C: usize,
    const MAX_S: usize,
    const D0: usize = { delay_size(MAX_S, 0.125) },
    const D1: usize = { delay_size(MAX_S, 1.0) },
    const D2: usize = { delay_size(MAX_S, 0.5) },
> {
    _phantom: core::marker::PhantomData<[(); MAX_S]>,
    buf0: [f32; D0],
    buf1: [f32; D1],
    buf2: [f32; D2],
}
```

Usage:

```rust
Reverb::<8, 9600>::new()                          // use defaults
Reverb::<8, 9600, 1200, 9600, 4800>::new()        // override
```

The struct and *every* impl block become free of `[(); EXPR]:` bounds. The compiler resolves defaults at instantiation.

**`f32` cannot be a const generic parameter type** (no `ConstParamTy`). But `f32` arithmetic works fine *inside* `const fn` returning `usize` defaults.

### Pattern 9 — `PhantomData` anchoring for unused const generics

When a const generic appears only in default-value expressions, the compiler reports `unconstrained generic constant`. Fix with `PhantomData`:

```rust
// ❌ MAX_S doesn't appear in any field — impl will fail
pub struct Foo<const MAX_S: usize, const D: usize = { MAX_S / 2 }> {
    buf: [f32; D],
}

// ✅ PhantomData anchors MAX_S at zero cost
pub struct Foo<const MAX_S: usize, const D: usize = { MAX_S / 2 }> {
    _phantom: core::marker::PhantomData<[(); MAX_S]>,
    buf: [f32; D],
}

Self { _phantom: core::marker::PhantomData, buf: [0.0; D] }
```

Use `PhantomData<[(); MAX_S]>` (zero-sized), not `[(); MAX_S]` directly (would occupy memory).

## Decision flow

```
Is the parameter fixed for the type's lifetime?
├── Yes → Struct-level const generic (Pattern 1)
│   └── Is any field size derived from this param?
│       ├── Simple expression → [(); EXPR]: where bound (Pattern 3)
│       ├── Complex formula → const fn helper (Pattern 4)
│       └── Multiple derived sizes with standard ratios?
│           └── Yes → Const generic defaults (Pattern 8)
│               └── Base param unused in fields? → PhantomData anchor (Pattern 9)
└── No, varies per call → Method-level const generic (Pattern 2)

Need a Vec of differently-parameterized items?
├── Known set of values → Enum dispatch (Pattern 6)
└── Open-ended → Consider dyn Trait instead

Array too large for stack?
└── Yes → Box<[T; N]> (Pattern 7)
```

## Checklist before reaching for const generics

1. **Do you actually need compile-time sizes?** If the size is truly dynamic (user input, file-dependent), use `Vec`. Const generics are for structurally fixed dimensions.
2. **Nightly required?** Basic `<const N: usize>` is stable. Arithmetic (`generic_const_exprs`) requires nightly.
3. **Monomorphization cost acceptable?** Each unique combination generates a new code copy. For 2–3 parameters with small value ranges, this is fine. For combinatorial explosions, weigh perf vs binary size.
4. **Where-bounds getting unwieldy?** If >5 where bounds, consider const fn helpers (Pattern 4), defaults (Pattern 8 + 9), or intermediate type aliases.
