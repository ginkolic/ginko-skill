# ginko-skill

A personal collection of agent skills for capturing everyday workflows. Skills follow the `SKILL.md` format, which works in Claude Code and other agents that support the Agent Skills standard (such as OpenAI Codex).

## Skills

The two `avoid-*` skills are designed to be used together; each covers a different failure mode of AI-generated code.

| Skill | Description |
| --- | --- |
| [avoid-overdefensive-code](skills/avoid-overdefensive-code) | Suppresses over-defensive programming: swallowed exceptions, needless null checks, fabricated defaults, silent fallbacks, compatibility shims. Includes a cleanup mode for reviewing diffs. Language references for Python, TypeScript, and Go. |
| [avoid-overengineering](skills/avoid-overengineering) | Suppresses over-engineering: premature abstractions, single-use helpers, unrequested config and feature flags, needless retries/async/caching, and logging/comment bloat. Includes a cleanup mode for reviewing diffs. |

## Repository Layout

```
skills/                  # installable skills, one directory each
└── <skill-name>/
    ├── SKILL.md         # required: frontmatter (name, description) + instructions
    ├── references/      # optional: detail loaded on demand
    ├── scripts/         # optional: executable helpers
    └── assets/          # optional: templates, images, other files
templates/
└── skill-template/      # starting point for new skills (not installed)
```

## Installation

### Claude Code

Copy a single skill:

```bash
cp -r skills/<skill-name> ~/.claude/skills/
```

Or symlink every skill so updates from `git pull` apply automatically:

```bash
mkdir -p ~/.claude/skills && for d in skills/*/; do ln -sfn "$PWD/$d" ~/.claude/skills/"$(basename "$d")"; done
```

For a single project, use `<project>/.claude/skills/` instead of `~/.claude/skills/`.

### OpenAI Codex

Codex reads the same `SKILL.md` format. Copy or symlink skills into the skills directory listed in the [Codex skills docs](https://developers.openai.com/codex/skills). The location has changed between versions, so check the docs for your version.

A skill only loads when the model decides it's relevant. For rules that should apply on every task, also copy the core principles into the project's `AGENTS.md`, which Codex reads on every run.

### ChatGPT (web / desktop)

ChatGPT doesn't load skill folders. Paste the body of `SKILL.md` into custom instructions or a project's instructions.

## Writing a New Skill

1. Copy the template: `cp -r templates/skill-template skills/<skill-name>`.
2. Set `name` to match the directory name (lowercase, hyphenated).
3. Write the `description` carefully. The model decides whether to load the skill from it alone, so state what the skill does, when to use it, and when not to.
4. Fill in the body, then add a row to the table above.

### Conventions

- **English** for all skill content.
- **Principles plus exceptions.** Every rule set needs a section on where the rules don't apply, or models over-correct.
- **End with a self-check** the model runs against its own output.
- **Keep `SKILL.md` short** (well under 500 lines). Move long examples and language- or domain-specific detail into `references/`, and say in `SKILL.md` when to read each file.
- **Grow from real failures.** Add rules and examples from mistakes you actually observe, not hypothetical ones.
