# LaraClaude Plugin - Development Guidelines

## Project Structure

```
laraclaude/
├── .claude-plugin/plugin.json    # Plugin metadata (name, author, version)
├── LICENSE                       # MIT
├── README.md                     # User-facing documentation with all skills listed
├── CLAUDE.md                     # This file - development guidelines
└── skills/                       # Each subdirectory is one skill
    └── skill-name/
        └── SKILL.md              # Skill definition (YAML frontmatter + instructions)
```

## Skill Naming Convention

All skills use the `lc:` prefix (short for LaraClaude):
- `/lc:skill-name` - invoked by user
- Lowercase, hyphen-separated
- Descriptive but concise

## SKILL.md Format

Every skill must follow this exact structure:

```yaml
---
name: lc:skill-name
description: One-line description (used by Claude to decide when to auto-invoke)
argument-hint: "[report | fix | fix --dry-run | target | fix target]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

# Skill Title

Brief overview paragraph.

## Subcommands

| Command | Description |
|---------|-------------|
| `/lc:skill-name` | Analyze and report |
| `/lc:skill-name fix` | Auto-fix with confirmation |
| `/lc:skill-name fix --dry-run` | Preview fixes without applying |
| `/lc:skill-name [target]` | Focus on specific file/model/table |
| `/lc:skill-name fix [target]` | Fix specific target |

## Step 1: ...
## Step 2: ...
```

## Subcommand Pattern

ALL analyzer skills must support these consistent modes:
- **No argument** = analyze and report only (never modify files)
- **`fix`** = auto-fix with confirmation before each change
- **`fix --dry-run`** = show what would be fixed without changing anything
- **`[target]`** = focus on specific file/model/table
- **`fix [target]`** = fix specific target

Generator skills (volt-component, generate-action, generate-modal, generate-table, generate-crud) don't need fix mode - they only create new files.

## Skill Categories

### Analyzers (report + fix)
Skills that scan existing code for issues. Must support all subcommands.
Examples: security-audit, find-n-plus-one, blade-audit, livewire-audit

### Generators (create only)
Skills that scaffold new files. Only accept a name/target argument.
Examples: volt-component, generate-action, generate-modal, generate-crud

### Hybrid
Skills that both analyze and can generate fixes (migrations, tests).
Examples: consolidate-migrations, generate-test

## Writing Skill Instructions

### Be Specific
- List EXACT patterns to grep for (regex when possible)
- Show EXACT output format with examples
- Specify EXACT file paths to check

### Be Safe
- Analyzer mode must NEVER modify files
- Fix mode must ask confirmation before each change
- Dry-run must show the diff without applying

### Be Laravel-Aware
- Support both class-based and Volt components
- Detect Docker containers from docker-compose.yml for artisan commands
- Use `@text()` pattern for translations (not `__()`)
- Understand morph maps in AppServiceProvider
- Know about Livewire 3 patterns (#[On], #[Computed], etc.)

### Be Project-Agnostic
- Don't hardcode project-specific paths
- Detect conventions from the project being analyzed
- Work with Laravel 8+ through 13+
- Handle both traditional Livewire and Volt components

## Report Format

All analyzer skills should output reports in this consistent format:

```
=== [Skill Name] Report ===

File: path/to/file.php
  Line 42: [SEVERITY] Description of issue
  → Suggestion or fix

File: path/to/other.php
  Line 15: [SEVERITY] Description
  → Suggestion

Summary: X issues found (Y critical, Z warnings)
```

Severity levels:
- `CRITICAL` - will cause errors/crashes
- `WARNING` - potential problems or bad practices
- `INFO` - suggestions for improvement

## Adding a New Skill

1. Create `skills/your-skill/SKILL.md`
2. Follow the YAML frontmatter format exactly
3. Include the Subcommands table
4. Add detailed step-by-step instructions
5. Update `README.md` with the new skill in the appropriate category
6. Test by running `/lc:your-skill` in a Laravel project

## Current Skills (27)

### Database & Migrations
- consolidate-migrations, check-foreign-keys, migration-fresh-test

### Models & Queries
- analyze-model, find-n-plus-one, orphaned-records, unused-columns

### Security & Performance
- security-audit, slow-queries, cache-opportunities

### Livewire & Blade
- livewire-audit, livewire-optimize, blade-audit

### Scaffolding
- volt-component, generate-action, generate-modal, generate-table, generate-crud, extract-action

### Testing
- generate-test, test-coverage

### Debug
- analyze-error

### Code Quality
- dead-code

### Documentation
- api-docs, model-diagram

### DevOps
- docker-check, deploy-checklist
