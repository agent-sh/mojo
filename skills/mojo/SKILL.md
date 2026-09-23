---
name: mojo
description: Use when writing, porting, reviewing or optimizing Mojo (.mojo files), including ownership and SIMD performance, GPU kernels and Python extension modules. Targets Mojo 1.1 and replaces pre-1.0 syntax. Not for pure Python or MAX serving.
allowed-tools: Read, Edit, Write, Grep, Glob, Bash(mojo:*), Bash(pixi:*)
---

# Mojo

Write Mojo that compiles on the current toolchain and performs well. Reference version: Mojo 1.1.0 (stable, 2026-09-17), with GPU programming in the MAX package. Docs: `mojolang.org/docs/` for the language and standard library, `max.modular.com` for GPU, `layout` and kernels.

Mojo changed fast through 2025 and 2026, and most Mojo in training data is older. Code that looks right from memory (`fn`, `inout`, `alias`, `@parameter if`, `UnsafePointer`, `from gpu import ...`) now fails to compile. The table below is the main thing this skill adds; the references hold the detail.

## Check the toolchain first

Run `mojo --version` (or read `pixi.lock`) before writing code, because the answer decides which column of the table applies. 1.0 shipped most renames with a deprecated alias and a compiler fix-it; 1.1 removed many of those aliases. On 0.26.x, several current spellings do not exist yet. When the installed compiler and this skill disagree, the compiler wins; say which version you targeted.

Stable APIs are marked per API in the reference docs; anything unmarked can still change between minor releases, so check it against the docs or the compiler when it matters.

## Old to current

| Old | Current (1.1) | Notes |
|---|---|---|
| `fn` | `def` | `fn` is an error since 1.0.0b2 and the keyword is gone in 1.1. `def` is the one static, checked function form. |
| `owned` / `borrowed` / `read` / `inout` args | `var` / `imm` (default) / `imm` / `mut` | `read` is a hard error in 1.1. Other conventions: `out` (constructors, named results), `ref` (parametric), `deinit` (destructors). |
| `x = 0` implicit declaration | `var x = 0` | An error in 1.1; walrus no longer declares. |
| `__copyinit__`, `__moveinit__` | `__init__(out self, *, copy: Self)`, `__init__(out self, *, deinit take: Self)` | Synthesized for `Copyable` / `Movable` types. Structs are `Movable` by default; opt out with `Movable where False`. |
| `__del__` | `__deinit__` | Destructor. `ImplicitlyDestructible` is now `Deinitable`. |
| `@value` | `@fieldwise_init` plus the traits you need | |
| `@register_passable(...)` | nothing, or `RegisterPassable` / `TrivialRegisterPassable` traits | Register passability is computed from contents. |
| `alias X = ...` | `comptime X = ...` | `alias` removed in 1.1. |
| `@parameter if` / `@parameter for` | `comptime if` / `comptime for` | Removed in 1.1. `@parameter` on closures became `@__parameter` (legacy closures only). |
| legacy closures passed as parameters | unified closures passed as arguments, with capture lists: `def body[w: Int](i: Int) {mut}: ...`, `lambda (x: Int) -> Int: x + 1` | `vectorize[simd_width](size, body)`, `sort`, `Bencher.iter`, `tile` take closures as runtime arguments. |
| `constrained(cond, msg)`, `__comptime_assert` | `comptime assert cond, msg`, or a trailing `where (cond, "msg")` | `where` inside a parameter list is gone; put it after the signature. |
| `from sys import ...`, `import math` | `from std.sys import ...`, `import std.math` | Implicit `std` imports are an error. Relative imports need `from . import x`. Same-package symbols need an explicit import (error in 1.1). |
| `UnsafePointer`, `ptr[i]`, `ptr + i`, `load()`, `store()`, `free()`, `memcpy` | `Pointer` with `unsafe_offset=i`, `unsafe_offset(i)`, `unsafe_load()`, `unsafe_store()`, `memory.alloc` with `Layout`, `unsafe_memcpy` | One pointer type; unsafety is on the operation. See `references/memory-and-performance.md`. |
| `InlineArray`, `StringSlice`, `CStringSlice` | `Array`, `StringSpan`, `CStringSpan` | `[1, 2, 3]` now builds an `Array`, not a `List`; write `var x: List = [1, 2, 3]` for a list. |
| `SIMD[dt, size=4]`, `.size` | `SIMD[dt, length=4]`, `.length` | `Int` is `Scalar[DType.int]`. 1.1 accepts `SIMD[.float64, 4]` where the type is known. |
| `NDBuffer`, stdlib `Tensor`, `DynamicVector` | `TileTensor` / `LayoutTensor` from `layout` (ships with MAX), `List`, `Array` | |
| `from gpu import ...`, `std.gpu` | `from max.gpu import ...`, `max.gpu.host`, `max.gpu.primitives` | `std.gpu` is private in 1.1. See `references/gpu.md`. |
| `mojo test` | `TestSuite.discover_tests[__functions_in_module()]().run()` in `main`, then `mojo run` | Test functions are `test_*` and pass unless they raise. |
| `mojo package`, `.mojopkg` | `mojo precompile`, `.mojoc` | `.mojopkg` support removed in 1.1. |
| `lst[-1]`, out-of-range slices | explicit indexes | Negative indexes are gone; an invalid slice aborts instead of clamping. |
| `magic` | `pixi`, or `uv` / `pip` / `conda` | |

## Rules that hold across tasks

- Copies are explicit for anything not `ImplicitlyCopyable`: `List`, `Dict`, `Array` and most user structs need `.copy()` or a move with `^`. This is what stops accidental duplication of large buffers; on hot paths borrow (`imm`, `ref`, `Span`) instead of copying.
- Structs are static: fields declared with `var` and a type, no inheritance, behavior shared through traits. Inside a struct, refer to its parameters as `Self.T`. A method's `self` has type `Self`; constrain with `where` instead of a custom self type.
- Functions do not raise unless marked `raises`, preferably with a type (`raises ParseError`). `abi("C")` functions (such as `PyInit_*`) cannot raise.
- Numeric variables do not convert implicitly: `Float32(n)`, `Int(u)`. Literals adapt to context.
- `min`, `max` are free functions in `std.math`; SIMD values have `.clamp()`, `.select()`, `.reduce_add()` and friends, and `.cast[DType.x]()`.
- Strings are UTF-8. `for c in s` yields grapheme clusters; use `byte_length()`, `count_codepoints()`, `[byte=i]`, `[codepoint=i:j]` or `bytes()` when you mean bytes or code points, which matters for tokenizers and parsers.
- References into `List`, `Dict`, `String` and similar carry an interior origin (experimental in 1.0); holding one across `append()` or `pop()` is a compile error rather than a dangling pointer. Re-fetch after mutating.

## References

Read the one that matches the task:

- `references/memory-and-performance.md`: pointers and allocation, ownership on hot paths, SIMD, `vectorize` / `parallelize`, benchmarking.
- `references/gpu.md`: `DeviceContext`, kernels, `TileTensor`, warps and shared memory, architecture dispatch.
- `references/python-interop.md`: calling Python from Mojo and building Python extension modules in Mojo.

Upstream sources, for anything these do not cover or that may have moved: changelogs at `mojolang.org/releases/` (append `.md` to any docs page for markdown), the standard library reference at `mojolang.org/docs/std/`, GPU docs at `max.modular.com/gpu/`, and source at `github.com/modular/modular`.

## Done

The code compiles with the installed `mojo` (`mojo run` or `mojo build`), tests pass through `TestSuite`, no construct from the left column remains unless the installed toolchain requires it, and any API you could not confirm for this version is named as unverified.
