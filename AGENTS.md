# Agent Instructions

This repository is the source of truth for managed AI skills. Installed skill directories are deployment targets, not editing locations.

When adding or adopting a skill:

1. Put managed skills under `skills/<name>`.
2. Ensure every managed skill has `skills/<name>/SKILL.md`.
3. Ensure `SKILL.md` starts with YAML frontmatter containing `name:` and `description:`.
4. Add one manifest line to `managed-skills.txt` in this format:

```text
skills/<name> claude,codex
```

Use `claude` or `codex` instead of `claude,codex` for agent-specific skills.

5. Run:

```sh
scripts/check
scripts/install --dry-run
```

6. Do not edit `~/.claude/skills` or `~/.codex/skills` directly.
7. Only run `scripts/install` when the user asks to install locally.
8. Keep scratch work in `experiments/`; do not add experiments to `managed-skills.txt`.

For Claude Code plugins, use `external/claude-plugins.txt` and `scripts/install-claude-plugins --dry-run`. Do not copy a full Claude plugin into `skills/` unless the user asks to adopt and maintain a specific skill from it.
