# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed
- Refreshed plugin/marketplace/Codex manifest descriptions to reflect the v0.2.0 scope: CPU/memory optimization and Mojo/Python interop, not just SIMD and GPU.

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
