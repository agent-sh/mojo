# mojo

A skill that teaches any coding agent to write idiomatic, correct, performant, current Mojo (Modular's language).

Mojo is young and fast-moving. The dominant failure mode for agents is emitting 2023-2024-era syntax that now warns or fails to compile. This plugin ships an always-on stale-to-current version map plus focused guidance for fundamentals, performance, and GPU - so agents target the current language (Mojo v1.0.0b1), not their training data.

## What it does

- Embeds a stale-to-current version map agents apply on every Mojo task: `def` not `fn`, `var`/`read`/`mut` not `owned`/`borrowed`/`inout`, `@fieldwise_init` not `@value`, synthesized `__init__(copy=)`/`__init__(take=)` not `__copyinit__`/`__moveinit__`, `TileTensor`/`LayoutTensor` not `NDBuffer`, `TestSuite` not the removed `mojo test` command.
- Core rules for idiomatic structs, traits, ownership, `comptime` metaprogramming, and typed error handling.
- Performance guidance: `SIMD`, `vectorize`/`parallelize`/`elementwise`, `@always_inline`, `LayoutTensor`, benchmarking.
- GPU guidance: `DeviceContext`, kernels, the thread/block/grid model, shared memory, `barrier()`, warps, occupancy (NVIDIA / AMD / Apple).
- Routes deep dives to a 41-source research guide in `agent-knowledge/`.

## Installation

```bash
claude plugin marketplace add agent-sh/mojo
claude plugin install mojo
```

Or add it through the agentsys marketplace alongside the rest of the agent-sh ecosystem.

## Usage

The skill activates automatically on Mojo work. Triggers include:

```text
write a Mojo struct for a 3D vector with SIMD ops
port this Python function to Mojo
is this idiomatic Mojo?
write a GPU vector-add kernel in Mojo
why is my Mojo parallelize loop slow?
```

It does not activate for Python, Rust, C++, or Modular MAX serving/deployment config.

## Layout

```
skills/mojo/SKILL.md                  the skill (version map + core rules + workflow)
agent-knowledge/
  mojo-best-practices-performance-gpu.md   deep-dive research guide (41 sources)
  resources/                          source metadata with confidence ratings
  README.md                           knowledge-base index
.claude-plugin/                       Claude Code plugin + marketplace manifests
.codex-plugin/                        Codex plugin manifest
```

## Development

```bash
agnix .                  # validate skill + manifests (zero errors required)
claude plugin validate . # validate marketplace and plugin manifests
```

Mojo guidance is grounded in current upstream docs at [mojolang.org/docs](https://mojolang.org/docs/), not model memory - re-verify any stdlib API outside stabilized-API markers against the installed toolchain.

## Related Plugins

- `skill-curator` - for writing and reviewing skills
- `agnix` - the linter that validates this plugin
- Part of the [agent-sh](https://github.com/agent-sh) ecosystem

## License

MIT
