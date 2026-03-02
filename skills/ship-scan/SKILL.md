---
name: ship-scan
user-invocable: true
description: |
  Scan the project codebase and auto-generate project-patterns and constraints.
  Analyzes ESLint, Prettier, tsconfig, package.json, file structure, and existing code
  to produce ready-to-use conventions.

  Usage: /ship-scan
  Also triggered by /ship-init after stack detection.

  Triggers: scan, analyze patterns, generate patterns, detect conventions
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, AskUserQuestion
---

# Project Scanner

**Scan codebase → Extract patterns → Generate constraints → Write project-patterns/SKILL.md**

---

## Step 1: Scan Config Files

Read all available config files in parallel:

```
# Linting & formatting
Read(file_path=".eslintrc.json")        # or .eslintrc.js, .eslintrc.yml
Read(file_path=".prettierrc")           # or .prettierrc.json, prettier.config.js
Read(file_path="biome.json")            # Biome config

# TypeScript
Read(file_path="tsconfig.json")

# Package info
Read(file_path="package.json")

# Editor config
Read(file_path=".editorconfig")

# Existing Claude config
Read(file_path=".claude/project.json")  # if exists
```

**Find config files dynamically:**
```
Glob(pattern=".eslintrc*")
Glob(pattern=".prettierrc*")
Glob(pattern="prettier.config.*")
Glob(pattern="biome.json")
Glob(pattern="tsconfig*.json")
```

## Step 2: Extract Lint Rules → Constraints

### ESLint
```
From .eslintrc:
  "max-lines": { max: 300 }         → constraint: "Files < 300 lines"
  "max-lines-per-function": 50      → constraint: "Functions < 50 lines"
  "complexity": 10                   → constraint: "Cyclomatic complexity < 10"
  "no-console": "error"             → constraint: "No console.log in production code"
  "import/order"                    → extract import order groups
  "@typescript-eslint/strict"       → constraint: "Strict TypeScript — no any"
  "react/jsx-no-bind"              → constraint: "No inline functions in JSX"
  "react-native/no-inline-styles"  → constraint: "No inline styles (use className/StyleSheet)"
```

### Prettier
```
From .prettierrc:
  "semi": false                     → pattern: "No semicolons"
  "singleQuote": true               → pattern: "Single quotes"
  "tabWidth": 2                     → pattern: "2-space indentation"
  "printWidth": 100                 → pattern: "Line width: 100 chars"
  "trailingComma": "all"            → pattern: "Trailing commas everywhere"
```

### TypeScript
```
From tsconfig.json:
  "strict": true                    → constraint: "Strict TypeScript mode"
  "noImplicitAny": true             → constraint: "No implicit any"
  "paths": { "@/*": ["./src/*"] }   → pattern: "Path alias: @/* → src/*"
  "baseUrl": "."                    → pattern: "Absolute imports from project root"
```

## Step 3: Analyze File Structure

```
# Get top-level structure
Bash(command="ls -d */")

# Count files per directory to understand architecture
Bash(command="find src -type f -name '*.ts' -o -name '*.tsx' -o -name '*.py' -o -name '*.go' | head -100")

# Detect common patterns
Glob(pattern="src/components/**/*.tsx")  → "Components in src/components/"
Glob(pattern="src/screens/**/*.tsx")     → "Screens in src/screens/"
Glob(pattern="src/hooks/**/*.ts")        → "Custom hooks in src/hooks/"
Glob(pattern="src/services/**/*.ts")     → "Services in src/services/"
Glob(pattern="src/store/**/*.ts")        → "State management in src/store/"
Glob(pattern="src/utils/**/*.ts")        → "Utilities in src/utils/"
Glob(pattern="app/**/*.tsx")             → "App Router structure (Next.js)"
Glob(pattern="pages/**/*.tsx")           → "Pages Router structure (Next.js)"
```

Build architecture description from what exists.

## Step 4: Analyze Code Patterns (Sample 3-5 Files)

Read a few representative files to extract actual patterns:

```
# Find the most recently modified files (likely current patterns)
Glob(pattern="src/**/*.tsx")  → pick 3 largest components
Glob(pattern="src/**/*.ts")   → pick 2 service files
```

For each sampled file, extract:
- **Import order** (group imports by type and extract the actual order)
- **Naming conventions** (PascalCase components? camelCase functions? kebab-case files?)
- **Export patterns** (default vs named)
- **State management** (useState, Zustand, Redux, Context?)
- **Styling approach** (Tailwind, CSS Modules, styled-components, NativeWind?)
- **Icon library** (which icon package is imported?)
- **Theme usage** (hardcoded colors vs theme tokens?)
- **Error handling** (try/catch patterns, error boundaries?)

## Step 5: Detect Dependencies → Tech Patterns

```
From package.json dependencies:
  "react-native"         → mobile app patterns
  "expo"                 → Expo-specific patterns (app.config, plugins)
  "next"                 → Next.js patterns (SSR, RSC, App Router)
  "zustand"              → "State: Zustand stores"
  "redux"                → "State: Redux + RTK"
  "@tanstack/react-query"→ "Data fetching: React Query"
  "drizzle-orm"          → "ORM: Drizzle (type-safe)"
  "prisma"               → "ORM: Prisma"
  "nativewind"           → "Styling: NativeWind (className)"
  "tailwindcss"          → "Styling: Tailwind CSS"
  "@tabler/icons-*"      → "Icons: Tabler only"
  "lucide-react"         → "Icons: Lucide"
  "@supabase/supabase-js"→ "Backend: Supabase"
  "firebase"             → "Backend: Firebase"
```

## Step 6: Generate Constraints

Combine all extracted rules into constraints:

```python
constraints = []

# From ESLint
if max_lines: constraints.append(f"Files < {max_lines} lines")
if no_console: constraints.append("No console.log in production code")

# From dependencies
if has_icon_lib: constraints.append(f"Icons: {icon_lib_name} only")
if has_styling: constraints.append(f"Styling: {styling_approach}")
if has_theme: constraints.append(f"Colors: Theme only, no hardcoded hex")
if has_state_mgmt: constraints.append(f"State: {state_lib} only")

# From code analysis
if import_order_detected: constraints.append(f"Import order: {detected_order}")
if naming_convention: constraints.append(f"Naming: {convention}")
```

## Step 7: Generate project-patterns/SKILL.md

```
Write(file_path=".claude/skills/project-patterns/SKILL.md", content=generated_content)
```

Generated content structure:

```markdown
---
name: project-patterns
description: "Auto-generated project conventions for {project_name}. Review and customize."
---

# Project Patterns — {project_name}

> Auto-generated by /ship-scan on {date}. Review and customize.

## Architecture
- **Structure:** {detected structure description}
- **Styling:** {NativeWind/Tailwind/CSS Modules/...}
- **State:** {Zustand/Redux/Context/...}
- **Data:** {React Query/SWR/fetch/...}
- **ORM:** {Drizzle/Prisma/none/...}

## File Conventions
- Components: `src/components/` — PascalCase files
- Screens: `src/screens/` — PascalCase files
- Hooks: `src/hooks/` — camelCase, `use` prefix
- Services: `src/services/` — PascalCase files
- Utils: `src/utils/` — camelCase files

## Import Order
```typescript
// 1. {detected group 1}
// 2. {detected group 2}
// ...
```

## Code Style
- {Prettier rules as conventions}
- {ESLint rules as conventions}

## Dependencies
- {key dependency}: {how it's used}

## Testing
- Test command: `{from package.json scripts}`
- Test location: `{detected test directory}`
- Pattern: {jest/vitest/pytest/...}
```

## Step 8: Update project.json Constraints

```
Read(file_path=".claude/project.json")

# Add detected constraints (don't overwrite existing)
Edit(file_path=".claude/project.json",
     old_string="\"constraints\": []",
     new_string="\"constraints\": [\n    {generated constraints as strings}\n  ]")

Read(file_path=".claude/project.json")  # verify
```

If constraints already exist → show diff and ask user:
```
AskUserQuestion:
  question: "Found new patterns. Merge with existing constraints?"
  options:
    - "Yes, merge all"
    - "Let me review each"
    - "Keep existing, skip"
```

## Step 9: Show Summary

```markdown
## Scan Complete!

### Detected Patterns
- Architecture: {summary}
- Styling: {approach}
- State: {library}
- Testing: {framework} at {location}

### Generated Constraints ({count})
- Files < 300 lines
- Icons: @tabler only
- Styling: NativeWind (className)
- No inline styles
- Strict TypeScript
- ...

### Files Updated
- `.claude/skills/project-patterns/SKILL.md` — project conventions
- `.claude/project.json` — constraints added

### Review Recommended
Please review the generated patterns and adjust as needed.
The more accurate your patterns, the better agents will follow your conventions.
```

## Integration With /ship-init

When init detects the stack and installs skills, it can auto-run scan:

```
After Step 9 (Create Starter Files) in ship-init:
  AskUserQuestion:
    question: "Scan codebase to auto-generate project patterns and constraints?"
    options:
      - "Yes, scan now" (Recommended)
      - "Skip, I'll add manually"

  If yes → execute this skill's full flow
```
