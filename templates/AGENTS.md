# AGENTS.md — {{PROJECT_NAME}}

> {{PROJECT_DESCRIPTION}}

## Entry Point

**For ANY development task → invoke `ship-router` skill FIRST. Never bypass.**

If `.claude/project.json` is missing → router triggers `/ship-init` automatically.

## Available Commands

| Command | When |
|---------|------|
| `/ship-router` | Any dev task (auto-triggered) |
| `/ship-init` | Bootstrap project / search skills |
| `/ship-scan` | Auto-generate patterns from codebase |
| `/ship-sprint` | Parallel build with agent teams |
| `/ship-plan` | Plan + Codex validation |
| `/ship-design` | UI design iteration |
| `/ship-hotfix` | Production fast-path |
| `/ship-status` | View progress + runs |
| `/ship-doctor` | Health check |
| `/ship-pr` | Generate PR |

## Tech Stack

{{TECH_STACK}}

## Project Structure

```
.claude/
├── project.json          # Project config (source of truth)
├── memory/               # Session memory (auto-managed)
│   └── activeContext.md   # Current context
└── skills/               # Installed skills
    └── project-patterns/  # Project-specific conventions
docs/
├── plans/                # Feature plans (auto-generated)
├── decisions/            # Decision log + ADRs
│   └── DECISIONS.md      # Why we chose X over Y
├── flows/                # Mermaid flow diagrams
└── kanban/               # Task board (if enabled)
    └── BOARD.md          # Auto-synced from progress
```

## Key Files

- **Config:** `.claude/project.json` — project rules, constraints, agent settings
- **Memory:** `.claude/memory/activeContext.md` — read on session start
- **Patterns:** `.claude/skills/project-patterns/SKILL.md` — project conventions

## Workflow Features

| Feature | What It Does | Updated When |
|---------|-------------|-------------|
| Decision Log | Records "why we chose X" for technical decisions | During planning, hotfixes |
| Flow Diagrams | Mermaid diagrams of user journeys & data flows | During planning |
| Kanban Board | Visual task board (Backlog → Done) | After each workflow run |

- Config: `.claude/project.json` → `features` section
- To disable: set `"enabled": false` in config

## Workflow

1. Router reads config + memory
2. Detects work type (UI, backend, bug, docs, etc.)
3. Loads relevant skills for detected stack
4. Executes agent chain: **Coder → Tester → Reviewer**
5. Logs run + updates memory

## Constraints

{{CONSTRAINTS}}
