# Unsafe code policy

## Crate-level lints

Every crate sets:

```rust
#![warn(unsafe_op_in_unsafe_fn)]   // require explicit unsafe block inside unsafe fn
```

Don't `#![forbid(unsafe_code)]` blanket — it's too restrictive for any real systems work. Allow per-module:

```rust
#![allow(unsafe_code)]   // at the top of a module that justifies unsafe
```

## `// SAFETY:` comment template

**Every `unsafe` block** has a preceding `// SAFETY:` comment explaining the invariants the caller is upholding. No exceptions, no shortcuts:

```rust
let value = unsafe {
    // SAFETY: `ptr` is non-null (checked above), aligned to `align_of::<T>()`
    // (allocated via `Layout::new::<T>()`), points to a valid `T` (initialized
    // by `init()` before any reader sees it), and outlives this read
    // (lifetime tied to `self`).
    *ptr
};
```

Required structure:

1. **Non-null / aligned / valid** — what *kind* of validity is required.
2. **Where the precondition comes from** — which earlier line / which caller invariant.
3. **Lifetime / aliasing** — why no concurrent mutation will happen.

If you can't write all three, the unsafe is wrong or unjustified — find another approach.

## When `unsafe` is justified

In rough order of frequency:

1. **FFI** — calling C / system APIs. Always unsafe at the FFI boundary.
2. **Self-referential or unmoveable data** — `Pin`, intrusive lists, etc.
3. **Bit-level / `transmute` casts** — prefer `bytemuck::cast` (safe transmute with `Pod` / `Zeroable`) when possible.
4. **Manual allocation** — `Box::from_raw`, `Vec::from_raw_parts`. Usually a sign you should use `Vec`/`Box` normally.
5. **SIMD intrinsics** — only when `std::simd` (portable_simd) can't express the operation. Document why.
6. **Performance** — last-resort, **must have benchmark** showing the safe version is unacceptable.

## Miri policy

If a crate uses `unsafe`, it must pass:

```bash
cargo +nightly miri test
```

Add to CI. Miri catches UB that compiles, runs, and produces correct-looking output but is unsound.

## Patterns that are usually wrong

- `unsafe { mem::transmute::<&T, &U>(x) }` — almost always wrong. Use `bytemuck::cast_ref` (requires `Pod`) for safe variants, or restructure.
- `unsafe { ptr.read() }` without explicit lifetime story — usually `*ptr` or `std::ptr::read` after explicit checks is clearer.
- `unsafe impl Send for X` / `unsafe impl Sync for X` without `SAFETY` comment — every such impl needs a paragraph explaining the shared-state story.
- Using `unsafe` to silence a borrow-checker error — almost never the right answer. The borrow checker is pointing at a real issue.

## Prefer safe abstractions

The `bytemuck` and `bytes` crates (whitelisted) cover most cases where you'd otherwise reach for raw `transmute` or unchecked indexing. Use them.

For zero-copy parsing of bytes-to-struct, `bytemuck::cast_slice::<u8, MyStruct>(&buf)` is safer than `&*(buf.as_ptr() as *const MyStruct)`.
