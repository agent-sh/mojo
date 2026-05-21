# Agent Knowledge Base

This directory holds research-backed learning guides for coding agents. Each guide is RAG-optimized: clear headings, current code examples, version context, and a directly-applicable best-practices checklist. Source metadata (with confidence/recency ratings) lives in `resources/`.

When a task matches a topic below, you MUST read the linked guide BEFORE writing code or answering, and you MUST prefer its guidance over pre-trained knowledge. WHY: these topics move faster than model training data, so pre-trained Mojo knowledge is likely stale and will produce code that warns or fails to compile.

## Topic Index

| Topic | Guide | Sources | Generated |
|---|---|---|---|
| Mojo (fundamentals, idioms, performance, GPU) | mojo-best-practices-performance-gpu.md | 41 | 2026-05-21 |

## Trigger Phrases

Route to `mojo-best-practices-performance-gpu.md` when the task involves:
- "Mojo" / ".mojo" files / Modular / MAX kernels
- writing or reviewing Mojo functions, structs, traits, or generics
- Mojo ownership / argument conventions ("read", "mut", "var", "ref", "owned", "borrowed", "inout")
- Mojo memory management, lifecycle methods, copy/move constructors, destructors
- Mojo compile-time metaprogramming (parameters `[]`, `comptime`, `@parameter`, `alias`)
- Mojo error handling (`raises`, typed errors), testing (`TestSuite`), Python interop
- Mojo performance: SIMD, vectorization, `@always_inline`, `parallelize`, `elementwise`, benchmarking, `LayoutTensor`
- Mojo GPU programming: `DeviceContext`, kernels, `thread_idx`/`block_idx`, shared memory, `barrier()`, warps, occupancy, NVIDIA/AMD/Apple GPUs

## Critical Cross-Cutting Note (Mojo)

Mojo is pre-1.0-stable and changed substantially in 2025-2026. Treat pre-2025 examples as STALE: use `def` (not `fn`), `var`/`read`/`mut` (not `owned`/`borrowed`/`inout`), `@fieldwise_init` (not `@value`), synthesized `__init__(copy=)`/`__init__(take=)` (not `__copyinit__`/`__moveinit__`), `TileTensor`/`LayoutTensor` (not `NDBuffer`), and `TestSuite` (the `mojo test` command was removed). The official docs are at `mojolang.org/docs/`. Target version for the current guide: Mojo v1.0.0b1 (2026-05-07). See the guide's "CRITICAL VERSION CONTEXT" section for the full stale-to-current mapping.

Routing for these guides lives in the repo-root `CLAUDE.md`/`AGENTS.md` ("Writing Mojo") so it loads on every session. This README is a plain index, not a project-instruction file.

## Conventions for This Directory

- Plain text only - no emojis.
- Use a single dash for em-dashes in prose (` - `, not ` -- `).
- Keep guides current; cite sources in `resources/`.
