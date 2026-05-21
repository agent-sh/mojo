---
name: mojo
description: "Use when writing, porting, reviewing, or optimizing Mojo (.mojo files): idiomatic syntax, CPU/memory perf (ownership, SIMD, vectorize/parallelize), GPU kernels (DeviceContext, TileTensor/LayoutTensor), and Mojo/Python extension modules. Targets v1.0.0b1, blocks stale pre-2025 syntax (fn, owned/inout, @value, NDBuffer). Not for pure Python, Rust, or MAX serving/deployment."
allowed-tools: Read, Edit, Write, Grep, Glob, Bash(mojo:*), Bash(pixi:*)
---

# Mojo

Write idiomatic, correct, current Mojo. Target version: Mojo v1.0.0b1 (2026-05-07). Canonical docs: `mojolang.org/docs/`.

Mojo is young and fast-moving. The dominant failure mode is emitting 2023-2024-era syntax that now warns or fails to compile. The version table below is the single most important asset - apply it on every Mojo task. The second failure mode (this skill's other focus): correct-looking high-performance code that compiles but silently copies, mis-types GPU intrinsics, or breaks the extension-module ABI - see "SOTA blind spots".

## CRITICAL VERSION CONTEXT - apply first

When you see the LEFT column in code, examples, or your own memory, it is STALE - use the RIGHT column.

| Stale (avoid) | Current (use) | Notes |
|---|---|---|
| `fn` keyword | `def` | `fn` warns now (slated to become a compile error). Write top-level and method functions as `def`. Nested `@parameter fn` closures still use `fn`. |
| `owned arg` | `var arg` | `var` means the function takes ownership. |
| `borrowed arg` | `read arg` (or omit - it is the default) | Immutable reference convention. |
| `inout arg` | `mut arg` | Mutable reference convention. |
| `__copyinit__` | `__init__(out self, *, copy: Self)` | Keyword-only `copy`. Auto-synthesized for `Copyable` types. |
| `__moveinit__` | `__init__(out self, *, deinit take: Self)` | Keyword-only `deinit take`. Auto-synthesized for `Movable` types. |
| `@value` | `@fieldwise_init` (+ derive `Copyable`, `Movable`) | `@value` superseded. |
| `@register_passable` / `@register_passable("trivial")` | `RegisterPassable` / `TrivialRegisterPassable` trait | Decorators removed; conform to the trait instead. |
| `alias X = ...` | `comptime X = ...` | `comptime` is the modern compile-time value keyword. |
| `@parameter if` / `@parameter for` | `comptime if` / `comptime for` | Compile-time branch/loop use `comptime`. `@parameter` decorator still valid on nested closures. |
| `from sys import ...`, `from memory import ...` (stdlib) | `from std.sys import ...`, `from std.memory import ...` | Stdlib lives under the `std` package (`std.sys`, `std.memory`, `std.algorithm`, `std.gpu`, `std.python`, ...). `layout` is its own package, NOT `std.layout`. |
| `constrained(cond, msg)` | `comptime assert cond, msg` | Must sit inside a function body, not at module/struct scope. |
| `NDBuffer` | `TileTensor` / `LayoutTensor` | `NDBuffer` removed entirely. |
| `DynamicVector` / `InlinedFixedVector` / stdlib `Tensor` | `List` / `InlineArray` / (no stdlib Tensor - use SIMD/List/UnsafePointer) | Removed collection types. |
| `mojo test` CLI | `TestSuite` struct + `mojo run test_file.mojo` | `mojo test` removed 2025-10-31. |
| `from pkg import x` binds `pkg` | binds only `x` | Import semantics tightened. |
| `gpu.block.sum(...)` | `gpu.primitives.block.sum(...)` | Collective ops moved under `gpu.primitives`. |
| `thread_idx` as `UInt` | `thread_idx` as `Int` | ID accessors migrated `UInt -> Int`. |
| `lst[-1]` negative index | not supported on stdlib collections | Index from front; CPU bounds-checking is on by default. |
| `docs.modular.com/mojo/` | `mojolang.org/docs/` | Docs domain moved (301 redirects active). |

When asserting anything that may be unstable, qualify with the version (e.g., "as of v1.0.0b1"). Stdlib outside stabilized-API markers can still change - re-verify against the installed toolchain.

Version sensitivity: the toolchain in use may lag this target. Stable 0.26.x still compiles `fn` (with a warning) and some lifecycle/keyword forms differ from v1.0.0b1. Always check `mojo --version` and prefer what the installed compiler accepts over this table when they disagree.

## When to use

Trigger on: writing/porting/reviewing/optimizing `.mojo` code; Mojo structs, traits, generics, ownership, lifecycle, `comptime`/`@parameter`; Mojo error handling, testing, Python interop; Mojo performance (SIMD, `vectorize`, `parallelize`, `benchmark`); Mojo GPU kernels.

Skip unless: the task names Mojo / Modular / MAX kernels, or touches a `.mojo` file. Do NOT activate for pure Python, Rust, C++, or MAX serving/deployment config.

## Core rules (apply on every task)

Language and syntax
- Use `def`. Functions are non-raising by default; add `raises` (prefer typed: `raises MyError`) to propagate errors.
- Argument conventions: `read` (default, immutable ref), `mut` (mutable ref), `var` (takes ownership; pair with `^` at the call site), `ref` (parametric, advanced). Plus `out` (constructors/named results), `deinit` (destructors). `read`/`mut`/`out`/`var`/`ref`/`deinit` are reserved - never use as identifiers.
- `struct` is the workhorse: static, no inheritance, fields declared with `var` + explicit type, initialized in `__init__`. Share behavior via traits (`Copyable`, `Movable`, `Writable`, ...), not inheritance. Use `@fieldwise_init` for the field-wise constructor. Inside a struct body, qualify own parameters as `Self.T` (bare `T` errors).
- Custom iterables signal end by RAISING, not by returning `Optional`: `def __next__(mut self) raises StopIteration -> Self.Element`. The collection conforms to `Iterable` and exposes a `comptime IteratorType[...]: Iterator`. Iterate with `for item in col:` (borrow) or `for ref item in col:` (mutate). `Dict` items iterate directly - `for e in d.items(): e.key, e.value` (no `[]` deref).
- Compile-time `[parameters]` vs run-time `(arguments)` drives metaprogramming. `comptime` for constants/values, branch selection (`comptime if`), and loop unrolling (`comptime for`). The `@parameter` decorator still applies to nested closures, but `@parameter if`/`@parameter for` are gone.
- Imports: stdlib modules are under the `std` package - `from std.sys import ...`, `import std.math`. The `layout` package (`LayoutTensor`, `TileTensor`) is separate, not under `std`.

## SOTA blind spots (what strong models get wrong)

These compile-fail, silently kill performance, or post-date model knowledge cutoffs. High value for an LLM inference/training engine optimized on CPU memory and GPU.

Memory and copies - the #1 silent perf killer
- Copies are NOT free and NOT implicit for most types. `List`, `Dict`, and user structs that conform only to `Copyable`/`Movable` (not `ImplicitlyCopyable`) require explicit `.copy()` or ownership transfer `^`. `var b = big_weights` or `return my_struct` errors until you write `^` or `.copy()` - this is the guardrail that stops accidental weight/KV-cache duplication. Move with `^` on last use; borrow with `read`/`ref`; never deep-copy a tensor on a hot path.
- `UnsafePointer` is non-null by design (null default ctor and `__bool__` deprecated). For nullable storage use `Optional[UnsafePointer[...]]` (same layout, `None` is the null niche). As a struct field it needs an explicit origin - use `MutExternalOrigin` for owned heap data: `var _ptr: UnsafePointer[Self.T, MutExternalOrigin]`. Allocate with the free function `alloc[T](count)` (returns `UnsafePointer`); `.free()` is manual.
- Pick the right pointer: `Pointer[T, mut, origin]` (safe, non-null view), `OwnedPointer[T]` (unique, Rust `Box`), `ArcPointer[T]` (refcounted shared), `Span(list)` (non-owning contiguous view - use for slices into weights/activations instead of copying).

CPU performance
- `min`/`max` are FREE FUNCTIONS in `std.math`, not SIMD methods: `from std.math import min, max; min(a, b)`. SIMD has methods `.clamp(lo, hi)`, `.select(t, f)` (per-lane via bool mask), `.reduce_add()/.reduce_max()/.reduce_min()`, `.cast[DType.x]()`.
- No implicit conversions between numeric *variables* - `Float32(my_int)`, `Int(my_uint)` explicitly. Literals ARE polymorphic and adapt to context (`var a: Float32 = 0.5`), so no wrapping needed for constants.
- Vectorize with `SIMD[dtype, size]` (`size` a power of two from `sys.simd_width_of[...]()`); raw vector loads via `ptr.load[width=N](idx)`. Use `std.algorithm` `vectorize`/`parallelize`/`elementwise` for data-parallel work, `@always_inline` on tiny hot functions, and parameterize over `DType`/sizes so the compiler monomorphizes. `prefetch` lives in `std.sys.intrinsics`.
- Strings are UTF-8: `len(s)` is byte length (deprecated on `String` - use `s.byte_length()` vs `s.count_codepoints()`). No slice syntax; byte access is `s[byte=i]` (returns `StringSlice`). Matters for tokenizer/vocab paths. `StaticString` is zero-allocation for compile-time constants.

GPU kernels

Skip unless: code targets an accelerator (DeviceContext, enqueue_*, kernels, TileTensor on device).
- `comptime assert tensor.flat_rank == N` is MANDATORY in any function that subscripts a `TileTensor` (kernel, host helper, CPU reference). Without it `tensor[r, c]` fails with `"invalid call to '__getitem__': lacking evidence to prove correctness"`. Derived tensors (`.tile()`, `.vectorize()`, `.distribute()`) get a NEW layout - re-assert `flat_rank` on the derived value before indexing.
- `enqueue_function` with a parameterized kernel: bind comptime params first, else a wall of "no matching method"/"DevicePassable" errors. `comptime k = vector_add[type_of(layout)]; ctx.enqueue_function[k](...)`. Monomorphic kernels (signature uses `type_of(layout)` directly) pass by name.
- Accumulating products from tensors with DIFFERENT layouts fails with `"cannot convert ElementType to Float32"` - `tensor[i]` returns `SIMD[dtype, layout_expr]` and the layout_exprs don't unify. Fix: `rebind[Scalar[dtype]](tensor[i])` per operand (builtin, no import). Not needed when all operands share one layout.
- Warp shuffle `offset`/`mask` args are `UInt32`, not `Int` - plain `Int` is a type-mismatch error. `warp.sum/max/min/broadcast/reduce` broadcast the result to every lane; `warp.shuffle_down/shuffle_xor` do not.
- `WARP_SIZE` is hardware-dependent (NVIDIA/AMD-RDNA = 32, AMD-CDNA = 64). Use `gpu.WARP_SIZE`, never hardcode 32. `global_idx.x`/`thread_idx.x` return `Int` - compare directly in bounds checks. Always `if tid < n:` (grid is rounded up). `barrier()` is block-wide only - never inside a divergent branch (deadlock).
- Architecture dispatch: `is_*` (`is_nvidia_gpu`, `is_amd_gpu`, ...) checks the COMPILATION TARGET - use inside kernels/GPU code. `has_*` (`has_accelerator`, `has_nvidia_gpu_accelerator`) checks the HOST - use from host code to decide whether to launch.
- Shared memory: `stack_allocation[dtype, address_space=AddressSpace.SHARED](row_major[M,N]())` from the `layout` package returns a `TileTensor`; the `std.memory` `stack_allocation` returns a raw pointer. `AddressSpace` is in `std.gpu.memory`. Async staging: `fragment.copy_from_async(src)` then `async_copy_wait_all()`. `Atomic.fetch_add(ptr, val)` from `std.atomic`.
- Profiling intuition: coalesce global-memory accesses; distrust high cache-hit rates (often means uncoalesced); target sufficient occupancy (25-50%), not maximum. Tensor cores are SM70+ (WMMA).

Mojo to/from Python (extension modules / serving glue)

Skip unless: code crosses the Python boundary (`PythonObject`, `PythonModuleBuilder`, `@export def PyInit_`, or a `.mojo` imported from Python).
- PythonObject -> Mojo scalar requires the `py=` KEYWORD: `Int(py=obj)`, `Float64(py=obj)`, `String(py=obj)`. Positional `Int(obj)` fails (only `Bool` accepts positional). Works on numpy scalars too.
- `Python.dict(...)` is generic over a single value type for all kwargs - mixed types fail to infer; wrap: `Python.dict(flag=PythonObject(b), count=PythonObject(42))`. Use `getenv` from `std.os` for env vars, not Python's `os`. No lambda - build callables via `Python.evaluate("lambda x: ...")`.
- Building an extension `.so`: `@export def PyInit_<name>() -> PythonObject` (name MUST match the `.mojo` filename), register with `PythonModuleBuilder` (`def_function` up to 6 args; `add_type[T]().def_py_init[...]().def_method[...]()`), compile `mojo build --emit shared-lib`. The shared-lib file must NOT contain a `main()` (`"error: shared library should not contain a 'main' function"`) - keep CLI code separate.
- Bound methods take either `py_self: PythonObject` (manual `py_self.downcast_value_ptr[Self]()`) or `self_ptr: UnsafePointer[Self, MutAnyOrigin]` (auto-downcast, direct field access). Return a Mojo value to Python with `PythonObject(alloc=value^)`; recover later via `downcast_value_ptr[T]()`. `import mojo.importer` auto-compiles `.mojo` on import (caches in `__mojocache__/`).
- Keep hot loops in Mojo - every `PythonObject` op crosses the FFI and allocates.

Ecosystem and versioning
- `magic` is dead - `pixi` fully replaced it for Mojo/MAX environments (or `uv`/`pip`/`conda` against `conda.modular.com`/`whl.modular.com`). Mojo needs a C linker (`gcc`/`clang`) to compile; Windows requires WSL2.
- MAX and Mojo versions must match major.minor when building custom MAX kernels - mismatch = kernel compilation failure. Keep both on the same channel (stable or nightly). Stable Mojo carries a `0.` prefix (e.g. `0.26.2`); MAX does not (e.g. `26.x`); `v1.0.0b1` is the 1.0 beta, ahead of current stable.

## Do NOT
- Do not emit any LEFT-column construct from the version table.
- Do not teach the old "`fn` is strict, `def` is dynamic" dichotomy - modern `def` carries the static, checked behavior.
- Do not use class inheritance, monkey-patching, or dynamic attributes on structs.
- Do not assume copies are free, that `UnsafePointer` can be null, or that `mojo test` exists.
- Do not hardcode warp size 32, skip the `flat_rank` assert, pass `Int` to warp shuffle, or call `enqueue_function` on an unbound parameterized kernel.

## Workflow
1. Confirm the target is Mojo and note the toolchain version (`mojo --version`, `pixi.lock`); reconcile against the version table's sensitivity note.
2. Apply the version table and core rules on every task. For possibly-unstable APIs (GPU tensor types, `vectorize`/`parallelize` signatures, `LayoutTensor` vs `TileTensor`, extension-module ABI), verify against current docs (see References) before committing to syntax.
3. For new code, prefer typed errors, explicit ownership (`^`/`.copy()`), and parameterization over `DType`/sizes. For ports, translate stale constructs via the version table first, then idiomatize, then optimize copies/SIMD/layout.
4. Verify: build/run with `mojo run` (or `mojo build`); write tests as `test_*` functions discovered by `TestSuite`.

## References
Verify any unstable or non-obvious API against current upstream docs (Mojo is young; stdlib outside stabilized-API markers still changes):

- Manual: https://mojolang.org/docs/manual/
- Stdlib reference: https://mojolang.org/docs/std/
- Releases / changelog (breaking changes by version): https://mojolang.org/releases/
- GPU: https://mojolang.org/docs/manual/gpu/fundamentals/ and the intro tutorial under the same path
- Ground-truth source (stdlib + MAX kernels): https://github.com/modular/modular
