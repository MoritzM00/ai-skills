# AI Skills

Personal source of truth for curated agent skills used across Claude Code and Codex.

This repository keeps a single managed skill collection:

- `skills/` contains every skill you choose to manage, whether you wrote it or adopted it from somewhere else.
- `experiments/` contains scratch work and is not installed by default.
- `managed-skills.txt` is the only list the installer reads.
- `external/claude-plugins.txt` lists Claude Code plugins to install through the Claude plugin marketplace.
- `external/README.md` explains external installs that are not managed as skills by this repo.

Installed directories such as `~/.claude/skills` and `~/.codex/skills` are deployment targets. Do not edit skills there directly; edit this repository and rerun `scripts/install`.

## Workflow

1. Create or copy a skill into `skills/<name>`.
2. Add the relative path and targets to `managed-skills.txt`.
3. Run `scripts/check`.
4. Run `scripts/install`.
5. Restart Codex if needed. Claude Code detects changes under watched skill directories.

Example manifest entry:

```text
skills/example-workflow claude,codex
```

The installed skill name is the final path segment. For `skills/example-workflow`, the install name is `example-workflow`.

## Adding Skills Later

For a new managed skill:

```sh
mkdir -p skills/<name>
# create skills/<name>/SKILL.md and any references/scripts/assets
printf 'skills/<name> claude,codex\n' >> managed-skills.txt
scripts/check
scripts/install --dry-run
scripts/install
```

Use `claude` or `codex` instead of `claude,codex` when the skill is agent-specific.

To have an agent do this, run it from this repository and give it a prompt like:

```text
Add a managed skill named <name> for <claude|codex|claude,codex>.
Put it under skills/<name>, create or update SKILL.md, add it to managed-skills.txt,
run scripts/check and scripts/install --dry-run, and summarize what changed.
Do not edit ~/.claude/skills or ~/.codex/skills directly.
Only run scripts/install if I explicitly ask.
```

For an adopted marketplace skill, use:

```text
Adopt the skill <source skill/path> into this repo as skills/<name>.
Preserve useful upstream source/license notes, remove assumptions that do not apply,
add the correct target in managed-skills.txt, then run scripts/check and scripts/install --dry-run.
Use claude,codex only if the skill is actually portable and tested in both agents.
```

## Commands

Check managed skills:

```sh
scripts/check
```

Preview installation:

```sh
scripts/install --dry-run
```

Install as symlinks:

```sh
scripts/install
```

Install as copies:

```sh
scripts/install --copy
```

Install for one target only:

```sh
scripts/install --target claude
scripts/install --target codex
```

Install external Claude Code plugins:

```sh
scripts/install-claude-plugins
```

Package one skill for Claude.ai upload:

```sh
scripts/package skills/example-workflow
```

ZIPs are written to `dist/`.

## Safety

The installer refuses to overwrite unrelated files or directories. It will replace:

- symlinks that already point inside this repository
- copied skill directories that contain this repository's `.ai-skills-managed` marker

Marketplace plugins stay outside this repository until you decide to manage one yourself. When that happens, copy the skill into `skills/<name>` and add it to `managed-skills.txt`.

## Agent-Specific Plugins

Some marketplace installs are full plugins, not just portable skill folders. Keep those in `external/claude-plugins.txt` if you want this repo to reinstall them through the Claude Code plugin marketplace.

For example, a Claude Code plugin installed with:

```text
/plugin marketplace add Imbad0202/academic-research-skills
/plugin install academic-research-skills
```

stays marketplace-managed because it may include Claude-specific plugin metadata, agents, commands, hooks, or assumptions. Do not add it to `managed-skills.txt` just because it is installed.

If you later want to own part of it, copy an individual skill folder into `skills/<name>`, review its references/scripts for Claude-only assumptions, and add it with the target it actually supports:

```text
skills/academic-paper claude
```

Only use `claude,codex` after adapting and testing the skill in both agents.
