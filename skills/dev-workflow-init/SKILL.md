---
name: dev-workflow-init
user-invocable: true
description: |
  Bootstrap a new project for dev-workflow. Auto-detects tech stack, recommends skills
  from skills.sh marketplace, generates project.json, and installs everything.

  Also triggered AUTOMATICALLY by the router when .claude/project.json is missing.

  Usage: /dev-workflow-init
  Or: just start working — router will trigger this on first run.

  Search for more skills: /dev-workflow-init search <query>
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion, WebFetch
---

# Project Bootstrap

**Detect stack → Confirm with user → Install skills from marketplace → Generate config → Ready.**

---

## Step 1: Check If Already Initialized

```
Read(file_path=".claude/project.json")
```

If file exists and user did NOT pass "search" argument → show current config summary. Offer to re-initialize or search for more skills.

If user passed "search <query>" → jump to **Skill Search** section.

## Step 2: Load Catalog

```
Read(file_path="{PLUGIN_DIR}/registry/catalog.json")
```

`{PLUGIN_DIR}` = the directory where this plugin is installed. Find it by locating this skill file's parent:
```
Bash(command="dirname $(dirname $(dirname $(readlink -f .claude/skills/dev-workflow-init/SKILL.md 2>/dev/null || echo .claude/skills/dev-workflow-init/SKILL.md)))")
```

If catalog not found → use hardcoded defaults (react-native, nextjs, python, go).

## Step 3: Auto-Detect Tech Stack

Scan the project root for stack indicators:

```
# Check for config files from catalog.stack_presets
for each stack in catalog.stack_presets:
  for each file in stack.detect.files:
    Glob(pattern=file)  # check if exists

# Check package.json dependencies (JS/TS projects)
Read(file_path="package.json")  # if exists
  → extract dependencies + devDependencies keys
  → match against stack.detect.package_deps

# Check for Python
Glob(pattern="requirements.txt")
Glob(pattern="pyproject.toml")

# Check for Go
Glob(pattern="go.mod")

# Check for Rust
Glob(pattern="Cargo.toml")
```

Build detected stacks list:
```
detected = []
for stack_name, stack_config in catalog.stack_presets:
  if any file matches OR any package_dep found:
    detected.append({
      name: stack_name,
      confidence: "high" if multiple signals else "medium",
      signals: [list of matched files/deps]
    })
```

## Step 4: Confirm With User

Present detection results and ask for confirmation:

```
AskUserQuestion:
  question: "Detected: {detected stacks}. Is this correct?"
  options:
    - "Yes, looks good"
    - "Add more" (→ ask what to add)
    - "Start from scratch" (→ ask for full stack)
```

If nothing detected:
```
AskUserQuestion:
  question: "Could not auto-detect tech stack. What are you building with?"
  options:
    - "React Native / Expo"
    - "Next.js / React"
    - "Python (FastAPI/Django/Flask)"
    - Other (free text)
```

## Step 5: Collect Project Info

```
AskUserQuestion:
  question: "What is the project name?"
  → default: directory name

AskUserQuestion:
  question: "Brief project description?"

AskUserQuestion:
  question: "Test command?"
  → default from catalog.stack_presets[stack].default_config.test_command
```

## Step 6: Select Skills

Based on detected stacks, show recommended + optional skills from catalog:

```
recommended = []
optional = []
for stack in detected_stacks:
  recommended.extend(catalog.stack_presets[stack].recommended_skills)
  optional.extend(catalog.stack_presets[stack].optional_skills)
recommended = deduplicate(recommended)
optional = deduplicate(optional) - recommended

# Show with install counts from catalog
AskUserQuestion:
  question: "Recommended skills for your stack:"
  multiSelect: true
  options:
    - "vercel-react-native (44K installs) — React Native performance" [pre-selected]
    - "ui-ux-pro-max — 50+ design styles, palettes" [pre-selected]
    - "expo-native-ui (14K installs) — Native UI components" [pre-selected]
    - "expo-deployment (7.9K installs) — EAS Build & deployment" [pre-selected]

AskUserQuestion:
  question: "Optional skills (popular for your stack):"
  multiSelect: true
  options:
    - "callstack-react-native (6.6K) — RN architecture from core contributors"
    - "expo-data-fetching (9.2K) — Native data fetching patterns"
    - "expo-tailwind (7.7K) — NativeWind setup"
```

**Also offer search:**
"Want to find more skills? I can search the skills.sh marketplace for anything."

## Step 7: Install Skills

**Primary method: `npx skills add` (from skills.sh marketplace)**

```
for skill in selected_skills:
  marketplace_id = catalog.skills[skill].marketplace
  if marketplace_id:
    Bash(command="npx skills add {marketplace_id}")
  elif catalog.skills[skill].fallback:
    # Bundled fallback — copy from registry
    PLUGIN_DIR = find_plugin_dir()
    Bash(command="cp -r {PLUGIN_DIR}/{fallback_path} .claude/skills/{skill_name}/")
```

**Install order:** Run all `npx skills add` commands. They auto-detect Claude Code and install to `.claude/skills/`.

**If npx fails** (no internet, npm not available):
```
# Fallback to bundled copies in registry/skills/
PLUGIN_DIR = find_plugin_dir()
for skill in selected_skills:
  if catalog.skills[skill].fallback:
    Bash(command="cp -r {PLUGIN_DIR}/{fallback_path} .claude/skills/{skill_name}/")
  else:
    warn("Skill {skill} requires internet to install. Skipping.")
```

## Step 8: Generate project.json

```
Bash(command="mkdir -p .claude")
```

Merge detected configs:
```python
config = {
  "version": "1.0",
  "project": {
    "name": user_project_name,
    "description": user_description,
    "tech_stack": detected_tech_list
  },
  "agents": {
    "tester": { "test_command": user_test_command },
    "retry_limits": { "tester_fail": 3, "reviewer_changes": 2 },
    "skip_conditions": {
      "tester": ["docs-only", "config-only", "user-says-skip"],
      "reviewer": ["docs-only", "config-only", "user-says-skip"],
      "designer": []
    }
  },
  "skills": {
    "project_patterns": ".claude/skills/project-patterns/SKILL.md",
    "auto_load": build_auto_load(installed_skills, catalog),
    "work_type_detection": merge_work_type_detection(detected_stacks, catalog)
  },
  "constraints": [],
  "integrations": {}
}
```

```
Write(file_path=".claude/project.json", content=JSON.stringify(config, null, 2))
```

## Step 9: Create Starter Files

```
# Memory directory
Bash(command="mkdir -p .claude/memory")

# Starter project-patterns (if not exists)
if not exists(".claude/skills/project-patterns/SKILL.md"):
  Bash(command="mkdir -p .claude/skills/project-patterns")
  PLUGIN_DIR = find_plugin_dir()
  Bash(command="cp {PLUGIN_DIR}/templates/project-patterns/SKILL.md .claude/skills/project-patterns/SKILL.md")
```

## Step 10: Show Summary

```markdown
## Project Initialized!

**Project:** {name}
**Tech Stack:** {tech_stack}
**Test Command:** `{test_command}`

### Installed Skills ({count})
| Skill | Source | Category |
|-------|--------|----------|
| vercel-react-native | skills.sh (44K installs) | UI |
| ui-ux-pro-max | bundled | UI |
| expo-native-ui | skills.sh (14K installs) | UI |
| supabase-postgres | skills.sh (26K installs) | Backend |

### Created Files
- `.claude/project.json` — project config
- `.claude/skills/project-patterns/SKILL.md` — add your conventions here
- `.claude/memory/` — memory directory (auto-populated on first run)

### Next Steps
1. Edit `.claude/skills/project-patterns/SKILL.md` with your project conventions
2. Add constraints to `.claude/project.json` (e.g. "Components < 300 lines")
3. Run `/dev-workflow-doctor` to verify setup
4. Start working! The router will handle everything.

### Find More Skills
- Search: `/dev-workflow-init search <query>`
- Browse: https://skills.sh/ | https://claudemarketplaces.com/
- CLI: `npx skills find <query>`
```

After summary, offer to scan codebase:
```
AskUserQuestion:
  question: "Scan codebase to auto-generate patterns and constraints?"
  options:
    - "Yes, scan now (Recommended)"
    - "Skip, I'll add manually"

If yes → invoke /dev-workflow-scan
```

---

## Skill Search (Available Anytime)

When user runs `/dev-workflow-init search <query>` or asks to find skills:

### Search via CLI
```
Bash(command="npx skills find {query} 2>&1")
```

Parse output to extract:
- Skill package IDs (e.g. `vercel-labs/agent-skills@vercel-react-native-skills`)
- Install counts
- URLs

### Present Results
```
AskUserQuestion:
  question: "Found {N} skills for '{query}'. Which to install?"
  multiSelect: true
  options:
    - "{skill-name} ({installs}) — {url}"
    - ...
```

### Install Selected
```
for skill in user_selected:
  Bash(command="npx skills add {skill_package_id}")
```

### Update project.json
After installing new skills, update `auto_load` in project.json:
```
Read(file_path=".claude/project.json")
# Add new skill to appropriate auto_load category
Edit(file_path=".claude/project.json", ...)
Read(file_path=".claude/project.json")  # verify
```

### Skill Search Examples
```
/dev-workflow-init search react-native    → Finds RN skills
/dev-workflow-init search supabase        → Finds Supabase skills
/dev-workflow-init search testing          → Finds testing skills
/dev-workflow-init search tailwind         → Finds CSS/Tailwind skills
/dev-workflow-init search expo             → Finds Expo-specific skills
/dev-workflow-init search python           → Finds Python skills
/dev-workflow-init search auth             → Finds authentication skills
```

---

## Auto-Detect Helpers

### build_auto_load(skills, catalog)
```python
auto_load = {"ui": [], "backend": []}
for skill_name in skills:
  skill_info = catalog.skills.get(skill_name)
  if not skill_info: continue
  category = skill_info.category
  if category in ("ui", "testing"):
    auto_load["ui"].append(skill_name)
  elif category == "backend":
    auto_load["backend"].append(skill_name)
return auto_load
```

### merge_work_type_detection(stacks, catalog)
```python
merged = {}
for stack in stacks:
  defaults = catalog.stack_presets[stack].default_config.get("work_type_detection", {})
  for work_type, config in defaults.items():
    if work_type not in merged:
      merged[work_type] = {"paths": [], "extensions": []}
    merged[work_type]["paths"].extend(config.get("paths", []))
    merged[work_type]["extensions"].extend(config.get("extensions", []))
# Deduplicate
for wt in merged:
  merged[wt]["paths"] = list(set(merged[wt]["paths"]))
  merged[wt]["extensions"] = list(set(merged[wt]["extensions"]))
return merged
```

## Re-Initialize Mode

When project.json already exists and user wants to re-init:
1. Read existing config
2. Show what will change
3. Confirm before overwriting
4. Preserve existing `constraints`, `integrations`, and `project_patterns`
5. Only update: `tech_stack`, `auto_load`, skills installation

## Skill Updates

To check for skill updates:
```
Bash(command="npx skills check 2>&1")
```

To update all installed skills:
```
Bash(command="npx skills update 2>&1")
```
