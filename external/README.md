# External Installs

This file explains external installs. The script-readable manifest for Claude Code plugins is `claude-plugins.txt`.

Use this file for context that does not belong in the machine-readable manifest.

## Why Keep This?

Some installs are agent-specific. For example, a Claude Code plugin can include plugin metadata, commands, agents, hooks, or Claude-specific assumptions. Those should stay installed through Claude Code until you decide to copy and maintain a specific skill yourself.

## Current External Installs

### Claude Code: academic-research-skills

Source:

```text
https://github.com/Imbad0202/academic-research-skills
```

Install commands, represented in `claude-plugins.txt`:

```text
/plugin marketplace add Imbad0202/academic-research-skills
/plugin install academic-research-skills
```

Local Claude plugin path:

```text
~/.claude/plugins/marketplaces/academic-research-skills
```

Status:

```text
Installed by scripts/install-claude-plugins, but not managed as a skill by this repo.
```

Reason:

```text
Claude Code plugin. Leave it marketplace-managed unless copying/adapting individual skills into ../skills/.
```

### Codex: academic-research-skills-codex

Source:

```text
https://github.com/Imbad0202/academic-research-skills-codex
```

Status:

```text
Not installed or managed by this repo.
```

Reason:

```text
Codex-native fork exists. Evaluate separately before adding any skill to ../skills/.
```

## Promotion Rule

If you want this repo to own a skill, copy it into:

```text
skills/<name>
```

Then add it to:

```text
managed-skills.txt
```

Example for a Claude-only adopted skill:

```text
skills/academic-paper claude
```
