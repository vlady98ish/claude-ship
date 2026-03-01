# dev-workflow

Generic agent orchestration plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Structured workflows with 7 specialized agents, config-driven everything, run observability, and DX commands.

> **One config file. Any project. Full quality pipeline.**

---

## Architecture Overview

```mermaid
graph TB
    User([User Request]) --> Router

    subgraph Router["Router (Entry Point)"]
        Config[Load project.json] --> Intent[Detect Intent]
        Intent --> Memory[Load Memory]
        Memory --> Skills[Load Skills]
        Skills --> Skip[Evaluate Skip Conditions]
    end

    Skip --> |ERROR| DEBUG
    Skip --> |MIGRATE| MIGRATE
    Skip --> |PLAN| PLAN
    Skip --> |REVIEW| REVIEW
    Skip --> |DEFAULT| BUILD

    subgraph BUILD["BUILD Workflow"]
        direction LR
        B_Des[Designer] -.->|UI only| B_Bld[Builder]
        B_Bld --> B_Tst[Tester]
        B_Tst --> B_Rev[Reviewer]
    end

    subgraph DEBUG["DEBUG Workflow"]
        direction LR
        D_Dbg[Debugger] --> D_Bld[Builder]
        D_Bld --> D_Tst[Tester]
        D_Tst --> D_Rev[Reviewer]
    end

    subgraph MIGRATE["MIGRATE Workflow"]
        direction LR
        M_Mig[Migrator] -.->|if app changes| M_Bld[Builder]
        M_Bld --> M_Tst[Tester]
        M_Tst --> M_Rev[Reviewer]
    end

    subgraph PLAN["PLAN Workflow"]
        direction LR
        P_Plan[Planner] --> P_Codex[Codex Validate]
        P_Codex -.->|issues found| P_Plan
    end

    subgraph REVIEW["REVIEW Workflow"]
        R_Rev[Reviewer]
    end

    BUILD --> MemUpdate[Memory Update]
    DEBUG --> MemUpdate
    MIGRATE --> MemUpdate
    PLAN --> MemUpdate
    REVIEW --> MemUpdate
    MemUpdate --> RunLog[Log to runs.jsonl]
    RunLog --> IntSync[Integration Sync]

    style Router fill:#4a90d9,color:#fff
    style MemUpdate fill:#27ae60,color:#fff
    style RunLog fill:#8e44ad,color:#fff
```

## Agent Contract Flow

Every agent produces a structured JSON contract. The router validates it before unblocking the next agent.

```mermaid
sequenceDiagram
    participant R as Router
    participant A as Agent
    participant V as Validator

    R->>A: Invoke with task + context + constraints
    A->>A: Execute task
    A->>R: Return output with JSON contract
    R->>V: Validate contract (required fields?)
    alt Valid
        V->>R: PASS
        R->>R: Unblock next agent
    else Invalid
        V->>R: FAIL (missing fields)
        R->>R: Create REMEDIATION task
        R->>A: Re-invoke with missing field list
    end
```

## Agents

```mermaid
graph LR
    subgraph "Read-Only (Diagnosis)"
        Debugger["Debugger\n(opus, red)"]
        Reviewer["Reviewer\n(opus, blue)"]
    end

    subgraph "Read-Write (Implementation)"
        Designer["Designer\n(sonnet, pink)"]
        Builder["Builder\n(sonnet, green)"]
        Tester["Tester\n(sonnet, yellow)"]
        Migrator["Migrator\n(opus, orange)"]
    end

    subgraph "Planning"
        Planner["Planner\n(opus, cyan)"]
    end

    Debugger -->|DIAGNOSIS| Builder
    Designer -->|GENERATED_CODE| Builder
    Builder -->|BUILDER_COMPLETE| Tester
    Tester -->|TEST_REPORT| Reviewer
    Reviewer -->|REVIEW_REPORT| Router([Router])
    Planner -->|PLAN_COMPLETE| Codex([Codex])
    Migrator -->|MIGRATION_PLAN| Builder

    style Debugger fill:#e74c3c,color:#fff
    style Designer fill:#e91e8e,color:#fff
    style Builder fill:#27ae60,color:#fff
    style Tester fill:#f1c40f,color:#000
    style Reviewer fill:#3498db,color:#fff
    style Planner fill:#1abc9c,color:#fff
    style Migrator fill:#e67e22,color:#fff
```

| Agent | Role | Contract Artifact |
|-------|------|-------------------|
| **Debugger** | Root cause analysis — REPRO, ROOT_CAUSE, FIX_HYPOTHESIS | `DIAGNOSIS` |
| **Designer** | UI generation via Gemini MCP | `DESIGNER_COMPLETE` |
| **Builder** | Implementation with TDD evidence | `BUILDER_COMPLETE` |
| **Tester** | Test execution with exit codes | `TEST_REPORT` |
| **Reviewer** | Security/pattern validation (read-only) | `REVIEW_REPORT` |
| **Planner** | Specs + task breakdown | `PLAN_COMPLETE` |
| **Migrator** | DB migrations with up/down + rollback | `MIGRATION_PLAN` |

## Workflow Chains

```mermaid
graph TD
    subgraph "BUILD (non-UI)"
        B1[Builder] --> B2[Tester] --> B3[Reviewer] --> B4[Memory]
    end

    subgraph "BUILD (UI)"
        U0[Designer<br/><i>Gemini MCP</i>] --> U1[Builder] --> U2[Tester] --> U3[Reviewer] --> U4[Memory]
    end

    subgraph "DEBUG"
        D0[Debugger] --> D1[Builder] --> D2[Tester] --> D3[Reviewer] --> D4[Memory]
    end

    subgraph "MIGRATE"
        M0[Migrator] -.-> M1[Builder] --> M2[Tester] --> M3[Reviewer] --> M4[Memory]
    end

    subgraph "HOTFIX (fast-path)"
        H0[Debugger] --> H1[Builder] --> H2[Tester<br/><i>targeted only</i>] --> H3[Memory + Postmortem]
    end

    subgraph "PLAN"
        P0[Planner] --> P1[Codex Validate] -.->|issues| P0
        P1 --> P2[Memory]
    end

    subgraph "DESIGN (standalone)"
        DS0[Designer<br/><i>Gemini MCP</i>] --> DS1[User Review] -.->|iterate| DS0
        DS1 --> DS2[Save + Memory]
    end

    subgraph "REVIEW"
        R0[Reviewer] --> R1[Memory]
    end

    style U0 fill:#e91e8e,color:#fff
    style D0 fill:#e74c3c,color:#fff
    style H0 fill:#e74c3c,color:#fff
    style M0 fill:#e67e22,color:#fff
    style DS0 fill:#e91e8e,color:#fff
    style P1 fill:#8e44ad,color:#fff
```

## Skills (Commands)

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `/dev-workflow-router` | Auto-detect intent, run full chain | Any dev task |
| `/dev-workflow-design` | Generate + iterate UI with Gemini | Before building UI |
| `/dev-workflow-plan` | Plan + Codex validation | Before major features |
| `/dev-workflow-hotfix` | Fast incident response | Production bugs |
| `/dev-workflow-status` | View tasks, runs, memory state | Check progress |
| `/dev-workflow-doctor` | Config, MCP, memory health check | Troubleshooting |
| `/dev-workflow-pr` | Generate PR from agent artifacts | After build completes |

## Memory System

```mermaid
graph LR
    subgraph ".claude/memory/"
        AC[activeContext.md<br/><i>focus, decisions, learnings</i>]
        PA[patterns.md<br/><i>gotchas, conventions</i>]
        PR[progress.md<br/><i>tasks, verification</i>]
        RL[runs.jsonl<br/><i>workflow history</i>]
    end

    subgraph "Archive"
        AR[archive/<br/><i>compacted entries</i>]
    end

    Router -->|load at start| AC
    Router -->|load at start| PA
    Router -->|load at start| PR
    Agents -->|memory_notes in contract| Router
    Router -->|persist at end| AC
    Router -->|persist at end| PA
    Router -->|log run| RL
    AC -.->|>50KB| AR
    PA -.->|>50KB| AR

    style AC fill:#e74c3c,color:#fff
    style PA fill:#3498db,color:#fff
    style PR fill:#27ae60,color:#fff
    style RL fill:#8e44ad,color:#fff
```

| File | Purpose | Anchors |
|------|---------|---------|
| `activeContext.md` | Current focus, decisions, learnings, references | `## Current Focus`, `## Decisions`, `## Learnings`, `## References` |
| `patterns.md` | Architecture, conventions, gotchas | `## Architecture Patterns`, `## Common Gotchas` |
| `progress.md` | Task tracking, verification evidence | `## Tasks`, `## Completed`, `## Verification` |
| `runs.jsonl` | Workflow execution log (append-only) | N/A (JSONL) |

**Hygiene:** Files > 50KB trigger compaction warning. Old entries archived to `.claude/memory/archive/`.

## Config (`project.json`)

```mermaid
graph TD
    Config[".claude/project.json"]

    Config --> Project["project{}<br/>name, description, tech_stack"]
    Config --> Agents["agents{}<br/>test_command, skip_conditions"]
    Config --> Skills["skills{}<br/>project_patterns, auto_load,<br/>work_type_detection"]
    Config --> Constraints["constraints[]<br/>freeform rules"]
    Config --> Integrations["integrations{}<br/>clickup, linear, etc."]

    Skills --> WTD["work_type_detection{}<br/>ui: paths + extensions<br/>backend: paths + extensions<br/>domain: paths"]
    Skills --> AL["auto_load{}<br/>ui: [skill1, skill2]<br/>backend: [skill3]"]

    Agents --> SC["skip_conditions{}<br/>tester: [docs-only, ...]<br/>reviewer: [docs-only, ...]"]

    style Config fill:#4a90d9,color:#fff
```

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

## Skip Conditions

```mermaid
flowchart TD
    Start[Agent in Chain] --> Check{Skip conditions<br/>match?}
    Check -->|No match| Run[Run Agent Normally]
    Check -->|docs-only| Skip
    Check -->|config-only| Skip
    Check -->|user-says-skip| Skip

    Skip[Skip Agent] --> Report["Produce SKIP_REPORT<br/>{artifact, agent, reason}"]
    Report --> Next[Next Agent Receives<br/>SKIP_REPORT instead<br/>of regular artifact]

    Run --> Contract["Agent Produces<br/>JSON Contract"]
    Contract --> Validate{Valid?}
    Validate -->|Yes| Next2[Unblock Next Agent]
    Validate -->|No| Remediate[REMEDIATION Task]
    Remediate --> Run

    style Skip fill:#f39c12,color:#fff
    style Report fill:#f39c12,color:#fff
    style Remediate fill:#e74c3c,color:#fff
```

## Quick Start

### 1. Install

```bash
git clone https://github.com/yourusername/claude-dev-workflow.git
```

### 2. Create `.claude/project.json`

Copy from `templates/project.json` and customize for your project.

### 3. (Optional) Add project patterns

Create `.claude/skills/project-patterns/SKILL.md` with your conventions. See `templates/project-patterns/SKILL.md`.

### 4. Add to your project's `CLAUDE.md`

```markdown
**For ANY development task → invoke `dev-workflow-router` skill FIRST. Never bypass.**
Memory: `.claude/memory/activeContext.md`
Config: `.claude/project.json`
```

### 5. Run `/dev-workflow-doctor` to verify setup

## Design → Build Pipeline

```mermaid
sequenceDiagram
    participant U as User
    participant D as /design
    participant G as Gemini MCP
    participant B as /router (BUILD)
    participant Bl as Builder

    U->>D: /dev-workflow-design settings page
    D->>G: Generate component
    G->>D: TSX code
    D->>U: Show code + constraints check
    U->>D: "Make the header bigger"
    D->>G: Regenerate with feedback
    G->>D: Updated TSX
    D->>U: Show updated design
    U->>D: "Looks good, save it"
    D->>D: Save to docs/designs/ + update memory

    Note over U,Bl: Later...

    U->>B: "Build the settings page"
    B->>B: Load memory → find design spec
    B->>Bl: Pass design spec to builder
    Bl->>Bl: Integrate with codebase
```

## Hotfix Flow

```mermaid
sequenceDiagram
    participant U as User
    participant R as /hotfix
    participant Dbg as Debugger
    participant Bld as Builder
    participant Tst as Tester

    U->>R: /dev-workflow-hotfix app crashes on login
    R->>Dbg: Diagnose
    Dbg->>Dbg: Reproduce → Isolate → Root Cause
    Dbg->>R: DIAGNOSIS {root_cause, fix_hypothesis}

    R->>Bld: Fix based on diagnosis
    Bld->>Bld: Minimal fix + rollback plan
    Bld->>R: BUILDER_COMPLETE {changes, rollback}

    R->>Tst: Verify fix (targeted tests only)
    Tst->>R: TEST_REPORT {exit: 0}

    R->>R: Generate postmortem
    R->>R: Update memory (patterns.md gotcha)
    R->>R: Log run to runs.jsonl
```

## Project Structure

```
claude-dev-workflow/
├── .claude-plugin/
│   └── plugin.json                   # Plugin manifest
├── agents/
│   ├── builder.md                    # Implementation specialist
│   ├── debugger.md                   # Root cause analysis
│   ├── designer.md                   # UI generation (Gemini MCP)
│   ├── migrator.md                   # DB migrations
│   ├── planner.md                    # Architecture & planning
│   ├── reviewer.md                   # Security & patterns (read-only)
│   └── tester.md                     # Quality assurance
├── skills/
│   ├── dev-workflow-router/SKILL.md  # Entry point & orchestration
│   ├── dev-workflow-memory/SKILL.md  # Memory management + hygiene
│   ├── dev-workflow-design/SKILL.md  # UI design iteration
│   ├── dev-workflow-plan/SKILL.md    # Manual plan + Codex
│   ├── dev-workflow-hotfix/SKILL.md  # Production fast-path
│   ├── dev-workflow-status/SKILL.md  # Workflow status + runs
│   ├── dev-workflow-doctor/SKILL.md  # Health diagnostics
│   └── dev-workflow-pr/SKILL.md      # PR automation
├── templates/
│   ├── project.json                  # Example config
│   └── project-patterns/SKILL.md    # Example patterns skill
├── CLAUDE.md                         # Router activation (3 lines)
└── README.md                         # This file
```

## Per-Project Files

| File | Purpose | Required? |
|------|---------|-----------|
| `.claude/project.json` | Project config | Yes |
| `.claude/skills/project-patterns/SKILL.md` | Project conventions | Optional |
| `.claude/memory/*.md` | Memory files | Auto-created |
| `.claude/memory/runs.jsonl` | Run history | Auto-created |
| `.claude/skills/{name}/SKILL.md` | Additional skills | Optional |
