---
name: dev-workflow-design
user-invocable: true
description: |
  Generate and iterate on UI designs before building.
  Uses Gemini MCP for code generation + project constraints for validation.
  Standalone — does NOT trigger builder/tester/reviewer.

  Triggers: /dev-workflow-design <description of what to design>
allowed-tools: Read, Write, Edit, Bash, Agent, Grep, Glob, TaskCreate, TaskUpdate, TaskList, TaskGet, AskUserQuestion
---

# Design Iteration Skill

**Generate UI code → Validate against constraints → User reviews → Iterate → Save.**

Unlike the router's automatic designer agent (inside BUILD chain), this skill is for **explicit design exploration** — iterate on UI without committing to the full build pipeline.

## Flow

### 1) Load Config + Context
```
Read(file_path=".claude/project.json")
Bash(command="mkdir -p .claude/memory")
Read(file_path=".claude/memory/activeContext.md")
Read(file_path=".claude/memory/patterns.md")
```

Extract: tech_stack, constraints, styling/icon/theme rules.

### 2) Load UI Skills
```
# Load project patterns
if config.skills.project_patterns:
  Read(file_path=config.skills.project_patterns)

# Load UI-specific skills from auto_load
for skill_name in config.skills.auto_load.get("ui", []):
  Read(file_path=".claude/skills/{skill_name}/SKILL.md")
```

### 3) Clarify Design Intent

Use `AskUserQuestion` if request is ambiguous:
- What screen/component to design?
- Any reference screens to match?
- Specific style preferences?
- Mobile, tablet, or responsive?

### 4) Gather Existing Patterns

Read similar existing components for style matching:
```
# Find existing components in the detected paths
Glob(pattern="src/components/**/*.tsx")  # or from config work_type_detection.ui.paths
# Read 2-3 similar components to extract style patterns
```

### 5) Generate with Gemini

Invoke designer agent with loaded context:

```
Agent(
  subagent_type="designer",
  prompt="
## Design Task
{user_request}

## Project Constraints
{config.constraints}

## Tech Stack
{config.project.tech_stack}

## Existing Component Examples
{code from similar components}

## UI Skills Loaded
{content from ui-ux-pro-max, vercel-react-native-skills, etc.}

---
Generate the component. Output your JSON contract with generated code.
Do NOT hand off to builder — this is design-only.
"
)
```

### 6) Validate Generated Code

Auto-check against project constraints:

| Check | Source | Pass/Fail |
|-------|--------|-----------|
| Correct icon library | constraints[] | ✓/✗ |
| Theme system used | constraints[] | ✓/✗ |
| Styling approach | constraints[] | ✓/✗ |
| File size limit | constraints[] | ✓/✗ |
| TypeScript types | auto | ✓/✗ |
| Import order | project-patterns | ✓/✗ |

If any check fails → auto-fix or flag for iteration.

### 7) Present to User

```
## Design: {component_name}

### Generated Code
\`\`\`tsx
{generated component code}
\`\`\`

### Target Path
`{suggested/file/path.tsx}`

### Constraints Check
- ✓ Icons: @tabler/icons-react-native
- ✓ Styling: NativeWind className
- ✓ Theme: useTheme() colors
- ✗ File size: 420 lines (limit: 400) — needs split

### Design Decisions
- {decision 1}: {rationale}
- {decision 2}: {rationale}
```

### 8) Iterate (User Feedback Loop)

If user wants changes:
1. Take feedback
2. Re-invoke designer with feedback + previous output
3. Re-validate
4. Present updated design

**Max iterations:** No limit — user controls when to stop.

### 9) Save Design Artifacts

When user approves:

**Option A: Save code to target path**
```
Write(file_path="{target_path}", content="{final_code}")
```

**Option B: Save as design spec (don't write code yet)**
```
Bash(command="mkdir -p docs/designs")
Write(file_path="docs/designs/YYYY-MM-DD-{component}-design.md", content="
# Design: {component}
## Code
{final code}
## Decisions
{design decisions}
## Constraints Met
{validation results}
")
```

User chooses: write code now, or save spec for later build.

### 10) Update Memory
```
Read(file_path=".claude/memory/activeContext.md")
Edit(file_path=".claude/memory/activeContext.md",
     old_string="## References",
     new_string="## References\n- Design: `{saved_path}`")
Read(file_path=".claude/memory/activeContext.md")  # Verify
```

Add design decisions to memory for builder to reference later.

## Gemini Fallback

If Gemini MCP is unavailable:
1. Use Claude (designer agent) to generate the code directly
2. Note in output: "Generated without Gemini — consider re-running when available"
3. Continue with validation and iteration as normal

## Relationship to BUILD Workflow

| Scenario | What to Use |
|----------|-------------|
| "Design me a settings page, let me review first" | `/dev-workflow-design` |
| "Build the settings page" (full chain) | Router → BUILD (auto-triggers designer if UI) |
| "I approved the design, now build it" | Router → BUILD (reads design spec from memory) |

The BUILD workflow checks `## References` in activeContext.md — if a design spec exists for the feature, builder uses it instead of starting from scratch.
