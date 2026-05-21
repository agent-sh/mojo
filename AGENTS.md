# mojo

> Skill that teaches any coding agent to write idiomatic, correct Mojo. Distributed via the agent-sh marketplace.

**Repository**: https://github.com/agent-sh/mojo

Follow the Karpathy Guidelines (simplicity, surgical changes, clear success criteria) when editing this plugin.

## Writing Mojo (read first)

Before writing, porting, or reviewing any Mojo, read `agent-knowledge/mojo-best-practices-performance-gpu.md` and prefer it over pretrained knowledge. WHY: Mojo changed substantially in 2025-2026, so pretrained Mojo syntax is likely stale and will warn or fail to compile. The shipped skill at `skills/mojo/SKILL.md` carries the always-on version map (`def` not `fn`, `var`/`read`/`mut` not `owned`/`borrowed`/`inout`, `@fieldwise_init` not `@value`, `TileTensor`/`LayoutTensor` not `NDBuffer`, `TestSuite` not `mojo test`). Target version: Mojo v1.0.0b1.

## Project Instruction Files

- `CLAUDE.md` is the project memory entrypoint for Claude Code.
- `AGENTS.md` is a byte-for-byte copy of `CLAUDE.md` for tools that read `AGENTS.md` (Codex CLI, OpenCode, Cursor, Cline, Copilot).
- Keep them identical. When editing one, update the other in the same commit.

## Critical Rules

1. **Plain text output** - No emojis, no ASCII art. Use `[OK]`, `[ERROR]`, `[WARN]`, `[CRITICAL]` for status markers.
2. **No unnecessary files** - Create only the files a task needs; skip summary, plan, audit, and temp docs unless required.
3. **Task is not done until it passes** - Every feature/fix needs quality tests, and `agnix` must be green before merge.
4. **Create PRs for non-trivial changes** - No direct pushes to main.
5. **Always run git hooks** - Let pre-commit and pre-push hooks run; never bypass them.
6. **Use single dash for em-dashes** - In prose, use ` - ` (single dash with spaces), never ` -- `. Does not apply to CLI flags like `--help`.
7. **Report script failures before manual fallback** - Surface broken tooling; never silently bypass it.
8. **Address all PR review comments** - Even minor ones. If you disagree, respond in the review thread.
9. **Wait for the claude workflow before merging** - It is the major quality gate and most thorough review; merge only after it passes.
10. **Follow the skill/command flow exactly** - If a flow specifies subagents, tools, or phases, follow them in order with no deviations.
11. **Token efficiency** - Save tokens over decorations.

## Skill Conventions

- `skills/mojo/SKILL.md` is the authoritative source of truth for how agents write Mojo; mirror any guidance added here into it.
- Keep the core skill short; route deep examples and reference material to `agent-knowledge/`.
- Prefer cross-tool features. Clearly gate any tool-specific behavior.
- Triggers should fire on realistic prompts ("write Mojo", "port this to Mojo", "is this Mojo idiomatic"). Use "Skip unless:" gates to avoid false activation.
- Ground Mojo guidance in current upstream docs, not memory - the language is young and changes fast. Cite sources for non-obvious rules.

## Model Selection

| Model | When to Use |
|-------|-------------|
| **Opus** | Complex reasoning, analysis, planning |
| **Sonnet** | Validation, pattern matching, most agents |
| **Haiku** | Mechanical execution, no judgment needed |

## Core Priorities

1. User DX (plugin users first)
2. Worry-free automation
3. Token efficiency
4. Quality output
5. Simplicity

## Testing

Before considering changes complete:
- Run `agnix` on the skill (zero errors).
- Verify the skill activates on realistic prompts in Claude Code and at least one other tool (Cursor or Codex recommended).

## References

- Part of the [agent-sh](https://github.com/agent-sh) ecosystem
- Mojo docs: https://mojolang.org/docs/ (canonical; old docs.modular.com Mojo URLs 301-redirect here)
- Research foundation: `agent-knowledge/mojo-best-practices-performance-gpu.md`
- https://agentskills.io
