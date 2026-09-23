# mojo

A skill that teaches any coding agent to write idiomatic, correct, performant, current Mojo (Modular's language).

Mojo is young and fast-moving, and most Mojo in model training data predates 1.0. The common failure is code that looks right from memory and no longer compiles. This plugin ships an old-to-current map for Mojo 1.1 plus short, verified references, so agents target the current language rather than their training data.

## What it does

- An old-to-current map applied on every Mojo task: `def` not `fn`, `var`/`imm`/`mut` not `owned`/`read`/`inout`, `comptime` not `alias` or `@parameter if`, unified closures and `lambda`, `Pointer` with `unsafe_` operations not `UnsafePointer`, `Array`/`StringSpan` renames, `max.gpu` not `std.gpu`, `TestSuite` not `mojo test`, `mojo precompile` not `mojo package`.
- Rules that hold across tasks: explicit copies and moves, struct and trait model, typed `raises`, explicit numeric conversion, grapheme-aware strings, interior origins.
- References for memory and CPU performance (pointers, allocation, SIMD, `vectorize`), GPU kernels (`DeviceContext`, `TileTensor`, warps, shared memory) and Python interop (bindings, extension modules).
- A toolchain check first: when the installed `mojo` is older or newer than 1.1, the compiler wins.

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
skills/mojo/SKILL.md     the skill (version map, core rules, doc links)
skills/mojo/references/  memory and performance, GPU, Python interop
.claude-plugin/          Claude Code plugin + marketplace manifests
.codex-plugin/           Codex plugin manifest
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
