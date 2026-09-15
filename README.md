# Thinkwise Skills for Claude Code

This repository contains a collection of [Claude Code](https://code.claude.com) **skills** for working with **Thinkwise Software Factory** models. Each skill is a reference guide or workflow that Claude loads automatically when a relevant task comes up — for example, creating a control procedure, setting up a cube, configuring a process flow, or reviewing a data model against Thinkwise naming conventions.

These skills are intended to be used together with an MCP connector that provides Software Factory access.

## What's in here

Each skill lives in its own folder and contains a `SKILL.md` file describing what it does and when Claude should use it, plus any supporting reference material it needs.

## Installing the skills

Claude Code looks for skills in two places:

| Scope | Path | Applies to |
|---|---|---|
| **Personal** | `~/.claude/skills/<skill-name>/` | All your projects |
| **Project** | `<your-project>/.claude/skills/<skill-name>/` | Just that one project |

To use these skills, copy (or symlink) each skill folder from this repo into one of those locations, keeping the folder name intact, e.g.:

```
~/.claude/skills/thinkwise-software-factory-cubes/SKILL.md
~/.claude/skills/thinkwise-software-factory-tasks/SKILL.md
...
```

- Use `~/.claude/skills/` if you want these skills available across **all** your projects.
- Use `<project>/.claude/skills/` if you only want them available in a **specific** project (useful if you want to commit them alongside a specific Software Factory repo).

No restart is required — Claude Code picks up new skills automatically the next time it evaluates which skills are relevant, or when you list available skills.

## Using a skill

Most of these skills are reference guides Claude invokes automatically based on their `description` when your request matches (e.g. asking it to create a cube, a process flow, or a control procedure). You generally don't need to invoke them by name — just describe what you want to do in your Software Factory model, and Claude will pull in the relevant skill before making changes.
