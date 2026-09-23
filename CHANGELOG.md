# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.3.0] - 2026-09-24

### Changed
- Retargeted from Mojo v1.0.0b1 to Mojo 1.1.0 (stable, 2026-09-17), verified against the 1.0.0b2, 1.0.0 and 1.1.0 changelogs, the 1.1 manual and standard library reference, and the MAX GPU docs.
- Rewrote the skill for current models: a short trigger description, the toolchain check, the old-to-current table, rules that hold across tasks, and a definition of done. Detail moved to `references/memory-and-performance.md`, `references/gpu.md` and `references/python-interop.md`. Removed the all-caps headings, the "Do NOT" list that restated the table, the workflow steps and the `Skip unless` gates.
- Refreshed plugin/marketplace/Codex manifest descriptions to reflect the v0.2.0 scope: CPU/memory optimization and Mojo/Python interop, not just SIMD and GPU.

### Fixed
- Stale or wrong for current Mojo: `read` (now `imm`; an error in 1.1), `UnsafePointer` and unprefixed pointer operations (now `Pointer` with `unsafe_*`), `alloc[T](count)` (now `Layout`-based `alloc`), `InlineArray`/`StringSlice` (now `Array`/`StringSpan`), `MutExternalOrigin` (now `MutUntrackedOrigin`), `std.gpu` imports (now `max.gpu`), `layout` as part of Mojo (now bundled with MAX), passing `Int` to kernels (no longer `DevicePassable`), the two-argument `enqueue_function` pattern, `@parameter` on closures (now `@__parameter`, legacy only), "no lambda" (Mojo has `lambda` since 1.0), iterating strings by code point (now graphemes), and `mojo package` (now `mojo precompile`).
- Warp shuffle masks are `UInt`, not `UInt32` (offsets are `UInt32`).
- `PyInit_*` exports need `abi("C")` and cannot raise.

## [0.2.0] - 2026-05-21

### Fixed
- Corrected three stale syntax rules, verified against mojolang.org v1.0.0b1 docs and working code on Mojo 0.26.2.0: stdlib imports use the `std.` package prefix (`from std.sys import ...`); compile-time control flow is `comptime if`/`comptime for` (not `@parameter if`/`@parameter for`); copy/move constructors are keyword-only (`__init__(out self, *, copy: Self)` / `__init__(out self, *, deinit take: Self)`).
- Fixed a dead stdlib reference URL (`/docs/stdlib/` -> `/docs/std/`).

### Added
- "SOTA blind spots" section - high-signal correction layer for high-performance Mojo (LLM inference/training on CPU memory and GPU), filtered to compile-fail, performance-killer, and post-knowledge-cutoff items: memory/copy semantics, pointer-type selection, CPU perf (SIMD, free-function `min`/`max`, explicit numeric conversion, string byte-indexing), GPU kernels (`flat_rank` assert, `enqueue_function` param binding, `rebind` for cross-layout accumulation, warp `UInt32` masks, `WARP_SIZE`, `is_`/`has_` dispatch, shared memory, async copy), Mojo/Python extension-module ABI, and ecosystem/versioning facts.
- Iterator protocol shape (`__next__` raises `StopIteration`, `Iterable`/`IteratorType` conformance) and `Dict.items()` direct iteration.

## [0.1.1] - 2026-05-21

### Changed
- The skill is now self-contained: removed the bundled `agent-knowledge/` deep-dive guide and instead link to current upstream docs (mojolang.org/docs, releases, github.com/modular/modular). Deep-dive material would drift from a fast-moving language; the online docs are the source of truth.

## [0.1.0] - 2026-05-21

### Added
- Initial release of the `mojo` skill plugin.
- `skills/mojo/SKILL.md` - teaches agents to write idiomatic, correct, current Mojo (target v1.0.0b1), with an always-on stale-to-current version map (`def` not `fn`, `var`/`read`/`mut` not `owned`/`borrowed`/`inout`, `@fieldwise_init` not `@value`, `TileTensor`/`LayoutTensor` not `NDBuffer`, `TestSuite` not `mojo test`), core rules for fundamentals, performance (SIMD/vectorize/parallelize), and GPU (DeviceContext, kernels, LayoutTensor).
- `agent-knowledge/mojo-best-practices-performance-gpu.md` - 41-source research foundation the skill routes to for deep dives.
- Claude Code and Codex plugin manifests; agnix and `claude plugin validate` CI.
