# AGENTS.md — Dev Workflow Plugin

> Universal ship plugin for Claude Code. Auto-detects stack, installs skills, routes tasks through agent chains.

## Entry Point

**For ANY development task → invoke `ship-router` skill FIRST. Never bypass.**

If `.claude/project.json` is missing → router triggers `/ship-init` automatically (stack detection + skill install).

## Available Commands

| Command | When |
|---------|------|
| `/ship-router` | Any dev task (auto-triggered) |
| `/ship-init` | Bootstrap new project |
| `/ship-init search <q>` | Find skills from marketplace |
| `/ship-scan` | Auto-generate patterns from codebase |
| `/ship-sprint` | Parallel build with agent teams |
| `/ship-plan` | Plan + Codex validation |
| `/ship-design` | UI design iteration |
| `/ship-hotfix` | Production fast-path |
| `/ship-status` | View progress + runs |
| `/ship-doctor` | Health check |
| `/ship-pr` | Generate PR |

## Config & Memory

- **Config:** `.claude/project.json` — project-specific rules (source of truth)
- **Memory:** `.claude/memory/activeContext.md` — read on session start
- **Patterns:** `.claude/skills/project-patterns/SKILL.md` — project conventions

## Workflow

1. Router reads config + memory
2. Detects work type (UI, backend, bug, docs, etc.)
3. Loads relevant skills for detected stack
4. Executes agent chain: **Coder → Tester → Reviewer**
5. Logs run + updates memory
