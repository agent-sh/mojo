---
name: mojo
description: "Use when writing, porting, reviewing, or optimizing Mojo (.mojo files): idiomatic syntax, SIMD/vectorize/parallelize performance, and GPU kernels (DeviceContext, LayoutTensor). Targets v1.0.0b1 and blocks stale pre-2025 syntax (fn, owned/inout, @value, NDBuffer). Not for Python, Rust, or MAX serving/deployment."
allowed-tools: Read, Edit, Write, Grep, Glob, Bash(mojo:*), Bash(pixi:*)
---

# Mojo

Write idiomatic, correct, current Mojo. Target version: Mojo v1.0.0b1 (2026-05-07). Canonical docs: `mojolang.org/docs/`.

Mojo is young and fast-moving. The dominant failure mode is emitting 2023-2024-era syntax that now warns or fails to compile. The version table below is the single most important asset - apply it on every Mojo task.

## CRITICAL VERSION CONTEXT - apply first

When you see the LEFT column in code, examples, or your own memory, it is STALE - use the RIGHT column.

| Stale (avoid) | Current (use) | Notes |
|---|---|---|
| `fn` keyword | `def` | `fn` warns now (slated to become a compile error). Write top-level and method functions as `def`. Nested `@parameter fn` closures still use `fn`. |
| `owned arg` | `var arg` | `var` means the function takes ownership. |
| `borrowed arg` | `read arg` (or omit - it is the default) | Immutable reference convention. |
| `inout arg` | `mut arg` | Mutable reference convention. |
| `__copyinit__` | `__init__(out self, copy: Self)` | Auto-synthesized for `Copyable` types. |
| `__moveinit__` | `__init__(out self, take: Self)` | Auto-synthesized for `Movable` types. |
| `@value` | `@fieldwise_init` (+ derive `Copyable`, `Movable`) | `@value` superseded. |
| `@register_passable` | conform to traits / `RegisterPassable` | Decorator removed. |
| `alias X = ...` | `comptime X = ...` | `comptime` is the modern compile-time value keyword. |
| `NDBuffer` | `TileTensor` / `LayoutTensor` | `NDBuffer` removed entirely. |
| `mojo test` CLI | `TestSuite` struct + `mojo run test_file.mojo` | `mojo test` removed 2025-10-31. |
| `from pkg import x` binds `pkg` | binds only `x` | Import semantics tightened. |
| `gpu.block.sum(...)` | `gpu.primitives.block.sum(...)` | Collective ops moved under `gpu.primitives`. |
| `thread_idx` as `UInt` | `thread_idx` as `Int` | ID accessors migrated `UInt -> Int`. |
| `lst[-1]` negative index | not supported on stdlib collections | Index from front; CPU bounds-checking is on by default. |
| `docs.modular.com/mojo/` | `mojolang.org/docs/` | Docs domain moved (301 redirects active). |

When asserting anything that may be unstable, qualify with the version (e.g., "as of v1.0.0b1"). Stdlib outside stabilized-API markers can still change - re-verify against the installed toolchain.

## When to use

Trigger on: writing/porting/reviewing/optimizing `.mojo` code; Mojo structs, traits, generics, ownership, lifecycle, `comptime`/`@parameter`; Mojo error handling, testing, Python interop; Mojo performance (SIMD, `vectorize`, `parallelize`, `benchmark`); Mojo GPU kernels.

Skip unless: the task names Mojo / Modular / MAX kernels, or touches a `.mojo` file. Do NOT activate for Python, Rust, C++, or MAX serving/deployment config.

## Core rules (apply on every task)

Language and syntax
- Use `def`. Functions are non-raising by default; add `raises` (prefer typed: `raises MyError`) to propagate errors.
- Argument conventions: `read` (default, immutable ref), `mut` (mutable ref), `var` (takes ownership; pair with `^` at the call site), `ref` (parametric, advanced). Plus `out` (constructors/named results), `deinit` (destructors).
- `struct` is the workhorse: static, no inheritance, fields declared with `var` + explicit type, initialized in `__init__`. Share behavior via traits (`Copyable`, `Movable`, `Stringable`, `Writable`, ...), not inheritance. Use `@fieldwise_init` for the field-wise constructor.
- Compile-time `[parameters]` vs run-time `(arguments)` drives metaprogramming. `comptime` for constants/values; `@parameter for`/`@parameter if` for unrolling/branch selection.

Correctness and safety
- Lifecycle via derived `Copyable`/`Movable` + synthesized `__init__(copy=)`/`__init__(take=)`. Never write `__copyinit__`/`__moveinit__`; never call lifecycle dunders directly. Structs are not copyable/movable by default - derive what you need.
- Move with `^`, borrow with `read`/`ref`, own with `var`. Rely on copy-to-move on last use, but verify hot paths.
- `Optional[UnsafePointer[...]]` for nullable pointers (`UnsafePointer` is non-null as of v1.0.0b1).
- Python interop: import whole modules inside functions (`Python.import_module("numpy")`), all values are `PythonObject`, keep hot loops in Mojo.

Performance
- Vectorize with `SIMD[dtype, size]` (`size` a power of two); pick width from `sys.simd_width_of[...]()`.
- Use `algorithm.elementwise`/`vectorize`/`parallelize` for data-parallel CPU work. `@always_inline` tiny hot functions. Parameterize over `DType`/sizes for specialization.
- Make layout explicit with `LayoutTensor`/`Layout.row_major(...)`; tile for locality. Benchmark with `benchmark.run[...]`, guard results with `keep(...)`.

GPU
- Drive allocation/launch/sync through `DeviceContext`: `enqueue_*` then `synchronize()`. Kernels are non-raising, return nothing, write into passed buffers/tensors.
- Bounds-check `if tid < n:` in every kernel (grid size is usually rounded up). Treat `thread_idx`/`block_idx`/`block_dim`/`grid_dim` as `Int`.
- Coalesce global-memory accesses; profile bandwidth, distrust high cache-hit rates. Stage tiles in shared memory; `barrier()` only block-wide, never inside a divergent branch (deadlock). Prefer warp ops within a warp. Use `gpu.WARP_SIZE`, never hardcoded 32. Target sufficient occupancy (25-50%), not maximum.

## Do NOT
- Do not emit any LEFT-column construct from the version table.
- Do not teach the old "`fn` is strict, `def` is dynamic" dichotomy - modern `def` carries the static, checked behavior.
- Do not use class inheritance, monkey-patching, or dynamic attributes on structs.
- Do not assume `mojo test` exists, or that copies are free.

## Workflow
1. Confirm the target is Mojo and note the toolchain version if discoverable (`mojo --version`, `pixi.lock`).
2. Apply the version table and core rules on every task. For possibly-unstable APIs (GPU tensor types, `vectorize`/`parallelize` signatures, `LayoutTensor` vs `TileTensor`), verify against current docs (see References) before committing to syntax.
3. For new code, prefer typed errors and explicit ownership. For ports, translate stale constructs via the version table first, then idiomatize.
4. Verify: build/run with `mojo run` (or `mojo build`); write tests as `test_*` functions discovered by `TestSuite`.

## References
Verify any unstable or non-obvious API against current upstream docs (Mojo is young; stdlib outside stabilized-API markers still changes):

- Manual: https://mojolang.org/docs/manual/
- Stdlib reference: https://mojolang.org/docs/stdlib/
- Releases / changelog (breaking changes by version): https://mojolang.org/releases/
- GPU: https://mojolang.org/docs/manual/gpu/fundamentals/ and the intro tutorial under the same path
- Ground-truth source (stdlib + MAX kernels): https://github.com/modular/modular
