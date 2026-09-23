# Memory and CPU performance

Read this for pointer work, allocation, ownership on hot paths, SIMD and parallel loops. Checked against the Mojo 1.0.0 and 1.1.0 changelogs and the 1.1 standard library reference.

## Pointers

- 1.0 unified `Pointer` and `UnsafePointer` into `Pointer`. Unsafe operations carry an `unsafe_` prefix or keyword: `ptr[unsafe_offset=i]`, `ptr.unsafe_offset(i)`, `unsafe_load()`, `unsafe_store()`, `unsafe_strided_load()`, `unsafe_gather()`, `unsafe_write(value^)`, `unsafe_write(copy=value)`, `unsafe_deinit_pointee()`, `unsafe_take_pointee()`, `unsafe_mut_cast()`. The unprefixed forms were deprecated in 1.0 and many were removed in 1.1. `UnsafePointer` still exists in 1.1 as a deprecated name.
- For a trivially deinitializable pointee (such as `Int`), `write()` is the safe counterpart to `unsafe_write()` (1.1).
- Aliases: `MutPointer`, `ImmPointer`, `OptionalPointer` (for nullable storage). Origins: `MutUntrackedOrigin` / `ImmUntrackedOrigin` for memory you manage yourself (the older `MutExternalOrigin` names are gone in 1.1), `ImmStaticOrigin` for static data. A struct field cannot hide `UnsafeAnyOrigin`; give the struct an `Origin` parameter or use an untracked origin. Discarding an origin is explicit: `as_unsafe_any_origin()`.
- Smart pointers: `OwnedPointer[T]` (unique; `into_inner()` to take the value), `ArcPointer[T]` (atomic refcount, with `WeakPointer`), `Span` (non-owning contiguous view, in `std.collections.span`; build from a raw pointer with `Span(unsafe_ptr=..., length=...)`).
- Raw memory functions are `unsafe_memcpy`, `unsafe_memset`, `unsafe_memset_zero`, `unsafe_memcmp`. `unsafe_memcpy` now checks exclusivity when `dest` and `src` come from the same buffer.

## Allocation

`std.memory.alloc` is layout-aware: `var a = alloc(Layout[Int32](count=n))` returns an owning `Allocation[T]`; free it with `dealloc(a^)` or use `ManagedAllocation`, which frees itself. `alloc[T](count)` without a `Layout` is deprecated; `unsafe_alloc` is the temporary spelling. `List.unsafe_take_allocation()` and `OwnedPointer.unsafe_take_allocation()` hand over an `Allocation`.

## Ownership on hot paths

- `var b = a` on a non-`ImplicitlyCopyable` value does not compile; write `a.copy()` or move with `a^` on the last use. Treat every `.copy()` of a large buffer as a design decision.
- Collections accept move-only elements (since 1.0.0b2); copy-requiring methods stay gated on `Copyable`. `for var x in list^:` consumes a list of move-only elements.
- `size_of()` returns the allocation size (stride) since 1.0, which matters for over-aligned types in manual layouts.

## SIMD and loops

- `SIMD[dtype, length]`, with `length` a power of two; `simd_width_of[dtype]()` from `std.sys` gives the host width. Loads and stores through a pointer: `ptr.unsafe_load[width=N](i)` / `unsafe_store`.
- `vectorize` (from `std.algorithm.functional`) takes the closure as an argument:

  ```mojo
  def body[width: Int](i: Int) {mut}:
      ...
  vectorize[simd_width](size, body)
  ```

  `parallelize` and `elementwise` live in the same package; closures move to runtime arguments release by release, so check the current signature before writing one.
- `@always_inline` still works; 1.1 adds `@inline(.always | .nodebug | .never | .automatic)`, which can depend on a parameter. Do not combine conflicting inline decorators (an error in 1.1).
- Prefetch options live in `std.sys.intrinsics` (`PrefetchOptions`).
- `--fp-mode=contract=off` disables FMA contraction when you need strict IEEE results (default is `fast`).

## Benchmarking

`std.benchmark`: `Bench.bench_function()` and `Bencher.iter()` take closures as runtime arguments in 1.1 (the parameter forms are gone). Use `keep()` to stop the optimizer from deleting the measured work.
