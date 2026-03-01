# CLAUDE.md — Dev Workflow Plugin

**For ANY development task → invoke `dev-workflow-router` skill FIRST. Never bypass.**

If `.claude/project.json` is missing → router triggers `/dev-workflow-init` automatically (stack detection + skill install).

Memory: `.claude/memory/activeContext.md` (read on session start)
Config: `.claude/project.json` (project-specific rules)

## Available Commands

| Command | When |
|---------|------|
| `/dev-workflow-router` | Any dev task (auto-triggered) |
| `/dev-workflow-init` | Bootstrap new project |
| `/dev-workflow-init search <q>` | Find skills from marketplace |
| `/dev-workflow-scan` | Auto-generate patterns from codebase |
| `/dev-workflow-sprint` | Parallel build with agent teams |
| `/dev-workflow-plan` | Plan + Codex validation |
| `/dev-workflow-design` | UI design iteration |
| `/dev-workflow-hotfix` | Production fast-path |
| `/dev-workflow-status` | View progress + runs |
| `/dev-workflow-doctor` | Health check |
| `/dev-workflow-pr` | Generate PR |
