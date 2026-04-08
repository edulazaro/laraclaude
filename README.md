# LaraClaude

A comprehensive Laravel toolkit plugin for [Claude Code](https://claude.ai/code). 10 specialized skills to analyze, debug, scaffold, and optimize Laravel applications directly from your terminal.

## Installation

```bash
/plugin install github:edulazaro/laraclaude
```

## Skills

### Database & Migrations

#### `/laraclaude:consolidate-migrations`
Analyze and consolidate fragmented migration files. When your project accumulates hundreds of small migrations (add column, modify column, rename...), this skill merges them into clean, optimized files.

```
/laraclaude:consolidate-migrations analyze        # Show consolidation report
/laraclaude:consolidate-migrations properties     # Consolidate a specific table
/laraclaude:consolidate-migrations all            # Consolidate all safe tables
```

**What it does:**
- Groups migrations by table
- Detects foreign key dependencies
- Classifies each as SAFE / CAUTION / DO NOT MERGE
- Merges ADD COLUMN into original CREATE
- Preserves data migrations
- Validates syntax after consolidation

#### `/laraclaude:check-foreign-keys`
Detect broken, orphaned, or missing foreign key constraints across your migrations and models.

```
/laraclaude:check-foreign-keys                    # Check all tables
/laraclaude:check-foreign-keys properties         # Check specific table
```

**Detects:**
- FK in migration but no relationship in model
- Relationship in model but no FK in migration
- FK referencing non-existent tables or columns
- Risky `onDelete` actions

#### `/laraclaude:migration-fresh-test`
Run `migrate:fresh` in your Docker container and get detailed error analysis with fix suggestions.

```
/laraclaude:migration-fresh-test                  # Run migrate:fresh
/laraclaude:migration-fresh-test --seed           # With seeders
```

### Models & Queries

#### `/laraclaude:analyze-model`
Deep analysis of any Eloquent model — relationships, scopes, casts, fillable, observers, keepers, actions, and potential issues.

```
/laraclaude:analyze-model Property                # Analyze specific model
/laraclaude:analyze-model                         # List all models
```

**Shows:**
- All relationships with types and related models
- Scopes, accessors, mutators
- Casts, fillable, guarded
- Observers, Larakeep keepers, Laractions actions
- Potential issues (missing fillable, N+1 risks)

#### `/laraclaude:find-n-plus-one`
Detect N+1 query problems in Blade views, Livewire components, and controllers.

```
/laraclaude:find-n-plus-one                       # Scan entire project
/laraclaude:find-n-plus-one resources/views/      # Scan specific directory
```

**Detects:**
- `$model->relationship` inside `@foreach` without eager loading
- Missing `with()` in Livewire computed properties
- Suggests the exact `with()` statement needed

#### `/laraclaude:orphaned-records`
Find database records where the parent (foreign key) no longer exists.

```
/laraclaude:orphaned-records                      # Check all tables
/laraclaude:orphaned-records operations            # Check specific table
```

#### `/laraclaude:unused-columns`
Detect database columns that are never referenced anywhere in the codebase.

```
/laraclaude:unused-columns                        # Check all tables
/laraclaude:unused-columns properties             # Check specific table
```

### Scaffolding

#### `/laraclaude:volt-component`
Generate a Livewire Volt single-file component following your project's conventions.

```
/laraclaude:volt-component UserProfile            # Create component
```

**Generates:**
- PHP logic at top with `new class extends Component`
- Proper mount(), validation, form methods
- `@text()` translations with English defaults
- Single root element, proper wire:model bindings

#### `/laraclaude:generate-action`
Create a Laractions action class with proper boilerplate and register it in the model.

```
/laraclaude:generate-action Property/ToggleFeatured
```

### Debug

#### `/laraclaude:analyze-error`
Paste a Laravel stacktrace and get root cause analysis with a specific fix.

```
/laraclaude:analyze-error
```

Then paste your error. The skill will:
- Parse the stacktrace
- Read the relevant source files
- Identify the root cause (not just the symptom)
- Provide a specific code fix

## Requirements

- [Claude Code](https://claude.ai/code)
- A Laravel project (8+, 9+, 10+, 11+, 12+)
- Docker (optional, for `migration-fresh-test` and `orphaned-records`)

## Contributing

Pull requests welcome. To add a new skill:

1. Create `skills/your-skill/SKILL.md`
2. Follow the YAML frontmatter format
3. Add to README
4. Submit PR

## License

MIT - see [LICENSE](LICENSE)
