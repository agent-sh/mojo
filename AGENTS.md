# mojo

> Skill that teaches any coding agent to write idiomatic, correct Mojo. Distributed via the agent-sh marketplace.

**Repository**: https://github.com/agent-sh/mojo

## Writing Mojo here

Before writing, porting or reviewing Mojo in this repo, apply `skills/mojo/SKILL.md` over what you remember: Mojo changed a lot in 2025-2026, so pretrained syntax is likely stale and fails to compile. Target version: Mojo 1.1.0. Check unstable APIs against https://mojolang.org/docs/.

## Conventions

- Output is plain text with the status markers `[OK]`, `[ERROR]`, `[WARN]`, `[CRITICAL]`, and no emojis or ASCII art.
- In prose, write a spaced single dash (` - `), not ` -- ` or an em dash. CLI flags like `--help` are fine.
- Put summaries, plans and audit notes in the PR or issue, not in committed files.
- Changes reach main through a PR. Keep git hooks on, and answer every review comment, in the thread when you disagree.
- When a script or tool fails, report the failure before working around it, so the tool gets fixed.
- When goals conflict, rank them: plugin users' experience, automation that needs no babysitting, token cost, output quality, simplicity.

## Skill conventions

- `skills/mojo/SKILL.md` is the source of truth for how agents write Mojo; mirror any guidance added here into it.
- Keep the skill file to the version map and the rules that apply to every task. Short, verified notes per area go in `skills/mojo/references/`; link to current upstream docs (mojolang.org, max.modular.com) for depth.
- Prefer cross-tool features, and mark any tool-specific behavior as such.
- Triggers should fire on realistic prompts ("write Mojo", "port this to Mojo", "is this Mojo idiomatic"). Keep the description short and name what it does not cover (pure Python, MAX serving).
- Ground Mojo guidance in current upstream docs, not memory, and cite sources for non-obvious rules: the language is young and changes fast.

## Testing

- `agnix --config .agnix.toml .` with zero errors, and `claude plugin validate .` for the manifests. CI runs both.
- For a trigger or description change, check that the skill activates on realistic prompts in Claude Code and one other tool (Cursor or Codex).
- Add a `CHANGELOG.md` entry under Unreleased.

## References

- Part of the [agent-sh](https://github.com/agent-sh) ecosystem
- Mojo docs: https://mojolang.org/docs/ (canonical; old docs.modular.com Mojo URLs redirect here)
- Mojo releases and changelog: https://mojolang.org/releases/
- https://agentskills.io
