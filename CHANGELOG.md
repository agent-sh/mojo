# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-05-21

### Added
- Initial release of the `mojo` skill plugin.
- `skills/mojo/SKILL.md` - teaches agents to write idiomatic, correct, current Mojo (target v1.0.0b1), with an always-on stale-to-current version map (`def` not `fn`, `var`/`read`/`mut` not `owned`/`borrowed`/`inout`, `@fieldwise_init` not `@value`, `TileTensor`/`LayoutTensor` not `NDBuffer`, `TestSuite` not `mojo test`), core rules for fundamentals, performance (SIMD/vectorize/parallelize), and GPU (DeviceContext, kernels, LayoutTensor).
- `agent-knowledge/mojo-best-practices-performance-gpu.md` - 41-source research foundation the skill routes to for deep dives.
- Claude Code and Codex plugin manifests; agnix and `claude plugin validate` CI.
