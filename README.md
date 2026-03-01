# dev-workflow

Generic agent orchestration plugin for Claude Code. Structured workflows with 7 specialized agents, config-driven everything, run observability, and DX commands.

## Agents

| Agent | Role | Model |
|-------|------|-------|
| **Designer** | UI code generation via Gemini MCP | sonnet |
| **Builder** | Implementation with TDD evidence | sonnet |
| **Tester** | Test execution with exit code evidence | sonnet |
| **Reviewer** | Security/pattern validation, confidence scoring | opus |
| **Planner** | Specs + task breakdown, Codex validation | opus |
| **Debugger** | Root cause analysis before fixes | opus |
| **Migrator** | DB migrations with rollback safety | opus |

## Workflows

| Trigger | Chain |
|---------|-------|
| build (UI) | designer → builder → tester → reviewer → memory |
| build (non-UI) | builder → tester → reviewer → memory |
| debug, fix, error | debugger → builder → tester → reviewer → memory |
| migrate, schema | migrator → [builder] → tester → reviewer → memory |
| review, audit | reviewer → memory |
| plan, design | planner → codex-validate → memory |
| /dev-workflow-hotfix | debugger → builder → targeted-tester → memory |

## Skills (Commands)

| Command | Purpose |
|---------|---------|
| `/dev-workflow-router` | Main entry point — auto-detects intent |
| `/dev-workflow-plan` | Manual plan + Codex validation |
| `/dev-workflow-hotfix` | Fast-path for production incidents |
| `/dev-workflow-status` | Show workflow state + run history |
| `/dev-workflow-doctor` | Diagnose plugin health |
| `/dev-workflow-pr` | Generate PR from workflow artifacts |

## Quick Start

### 1. Install

```bash
git clone https://github.com/yourusername/claude-dev-workflow.git
```

### 2. Create `.claude/project.json`

```json
{
  "version": "1.0",
  "project": {
    "name": "my-app",
    "description": "My project",
    "tech_stack": ["typescript", "next.js"]
  },
  "agents": {
    "tester": { "test_command": "npm test" },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"],
      "designer": []
    }
  },
  "skills": {
    "project_patterns": ".claude/skills/project-patterns/SKILL.md",
    "auto_load": { "ui": ["ui-ux-pro-max"], "backend": [] },
    "work_type_detection": {
      "ui": { "paths": ["components/"], "extensions": [".tsx"] },
      "backend": { "paths": ["api/"], "extensions": [".ts"] }
    }
  },
  "constraints": ["Components < 300 lines"],
  "integrations": {}
}
```

### 3. Add to CLAUDE.md

```
**For ANY development task → invoke `dev-workflow-router` skill FIRST. Never bypass.**
Memory: `.claude/memory/activeContext.md`
Config: `.claude/project.json`
```

## Key Features

### JSON Agent Contracts
Every agent outputs a structured JSON block. Router validates required fields before unblocking the next agent. No more fragile text parsing.

### Skip Conditions
Agents can be skipped via config. When skipped, router produces a `SKIP_REPORT` that replaces the agent's artifact.

### Run Observability
Every workflow logs to `.claude/memory/runs.jsonl`: workflow type, agents run, result, issues found. View with `/dev-workflow-status`.

### Memory Hygiene
Auto-warns when memory files exceed 50KB. Compaction protocol archives old entries. Scoped loading for large projects.

### Config Validation
Router validates `project.json` on load. Missing fields get defaults. Invalid config gets clear error messages. Run `/dev-workflow-doctor` for full health check.

### Hotfix Fast-Path
`/dev-workflow-hotfix` — shortened chain (no reviewer), mandatory rollback plan, auto-generated postmortem template.

## Project Structure

```
claude-dev-workflow/
├── .claude-plugin/plugin.json
├── agents/
│   ├── builder.md          # Implementation
│   ├── debugger.md         # Root cause analysis
│   ├── designer.md         # UI generation (Gemini MCP)
│   ├── migrator.md         # DB migrations
│   ├── planner.md          # Architecture
│   ├── reviewer.md         # Security & patterns
│   └── tester.md           # Quality assurance
├── skills/
│   ├── dev-workflow-router/    # Entry point
│   ├── dev-workflow-memory/    # Memory management
│   ├── dev-workflow-plan/      # Manual plan + Codex
│   ├── dev-workflow-hotfix/    # Production fast-path
│   ├── dev-workflow-status/    # Workflow status
│   ├── dev-workflow-doctor/    # Health diagnostics
│   └── dev-workflow-pr/        # PR automation
├── templates/
│   ├── project.json
│   └── project-patterns/SKILL.md
├── CLAUDE.md
└── README.md
```
