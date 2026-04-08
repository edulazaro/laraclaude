---
name: lc:analyze-model
description: Deep analysis of a Laravel Eloquent model - relationships, scopes, casts, fillable, observers, keepers, actions.
argument-hint: "[analyze | fix | fix --dry-run | ModelName | fix ModelName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Analyze Model

Perform a comprehensive analysis of a Laravel Eloquent model, extracting all relevant information about its structure, relationships, behaviors, and potential issues.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | List all models and ask which to analyze. Read-only report. |
| `fix` | Analyze all models and auto-fix common issues (missing fillable, broken casts, etc.) with confirmation before each change. |
| `fix --dry-run` | Show what fixes would be applied across all models without changing anything. |
| `[ModelName]` | Analyze a specific model. Read-only report. |
| `fix [ModelName]` | Analyze and fix issues in a specific model, with confirmation before each change. |

## Analysis Process

### Step 1: Locate the Model

1. If a `[ModelName]` argument is provided:
   - Search for the model in `app/Models/{ModelName}.php` using `Glob`.
   - If not found, try case-insensitive search across `app/Models/` directory.
   - If still not found, search the entire `app/` directory for a class with that name.
2. If no argument is provided:
   - Use `Glob` to list all files in `app/Models/*.php`.
   - Display the list of available models and ask the user which one to analyze.

### Step 2: Read and Parse the Model

Use `Read` to load the full model file. Extract the following sections:

#### A. Class Metadata
- **Namespace and class name**
- **Parent class** (Model, Authenticatable, Pivot, etc.)
- **Traits used** (HasFactory, SoftDeletes, HasKeepers, HasActions, HasSources, etc.)
- **Class-level attributes** (`#[KeptBy(...)]`, `#[UsesOrigin(...)]`, etc.)
- **Table name** (explicit `$table` property or conventional)
- **Primary key** (if customized)
- **Timestamps** (if disabled)
- **Connection** (if non-default)

#### B. Mass Assignment
- **$fillable** array -- list all fields
- **$guarded** array -- list all fields
- If neither is defined, flag as a potential security issue

#### C. Attribute Casting
- **$casts** array or `casts()` method -- list all casts with their types
- Note any enum casts and identify the enum class

#### D. Relationships
For each relationship method, extract:
- Method name
- Relationship type (belongsTo, hasMany, hasOne, belongsToMany, morphTo, morphMany, morphOne, morphToMany, hasManyThrough, hasOneThrough)
- Related model class
- Foreign key (explicit or conventional)
- For belongsToMany: pivot table, pivot columns, withTimestamps
- For morphTo/morphMany: morph name

Display as a structured list:
```
RELATIONSHIPS
==============
belongsTo:
  - organization()    -> Organization::class  (FK: organization_id)
  - office()          -> Office::class        (FK: office_id)
  - user()            -> User::class          (FK: user_id)

hasMany:
  - properties()      -> Property::class      (FK: client_id)
  - operations()      -> Operation::class     (FK: client_seller_id)

belongsToMany:
  - tags()            -> Tag::class           (pivot: client_tag, columns: [])
```

#### E. Scopes
- **Local scopes** (`scopeActive`, `scopeForUser`, etc.)
- **Global scopes** (applied in `booted()` or via attributes)
- Show the scope signature and a brief description of what it filters

#### F. Accessors and Mutators
- **Laravel 9+ style**: `Attribute::make(get: ..., set: ...)`
- **Legacy style**: `getXAttribute()`, `setXAttribute()`
- **Appended attributes**: `$appends` array
- For each accessor, note if it accesses relationships (potential N+1 risk)

#### G. Observers
- Search for an observer in `app/Observers/{ModelName}Observer.php`
- If found, list all observer methods (creating, created, updating, updated, deleting, deleted, etc.)
- Note what each observer method does (brief summary)

#### H. Larakeep Keepers
- Check for `#[KeptBy(KeeperClass::class)]` attribute
- If found, read the keeper class and list all `getX()` methods
- Show what attributes each keeper method populates

#### I. Laractions Actions
- Check for `$actions` array property
- List all registered actions with their class paths
- For each action, read the class and show:
  - The `handle()` method signature
  - Brief description of what it does

#### J. Additional Features
- **Boot methods**: `boot()`, `booted()` -- any event listeners or global scopes
- **Custom methods**: Public methods that are not relationships, scopes, or accessors
- **Constants**: Any class constants defined
- **Interfaces implemented**: Check implements clause

### Step 3: Cross-Reference with Migration

1. Find the migration that creates this model's table using `Grep` to search for `Schema::create('{table_name}'`.
2. Compare migration columns with model's `$fillable`:
   - Columns in migration but NOT in `$fillable` (and not in standard exclusions like id, timestamps, etc.)
   - Columns in `$fillable` but NOT in migration (possibly computed or from a different source)
3. Note column types from migration for reference.

### Step 4: Detect Potential Issues

Scan for these common problems:

1. **Missing fillable entries**: Columns exist in migration but are not in `$fillable` or `$guarded`.
2. **Relationships without foreign keys**: A `belongsTo` relationship where the FK column doesn't have a foreign key constraint in the migration.
3. **N+1 risk in accessors**: Accessors that access `$this->relationship` without the relationship being commonly eager-loaded.
4. **Missing casts**: Date columns without date casts, JSON columns without array/json casts, boolean columns without boolean casts.
5. **Soft deletes mismatch**: Model uses `SoftDeletes` trait but migration doesn't have `softDeletes()`, or vice versa.
6. **Missing observer registration**: Observer file exists but may not be registered in a service provider.

### Step 5: Generate Relationship Diagram

Create a text-based relationship diagram showing how this model connects to others:

```
                    +------------------+
                    |   Organization   |
                    +------------------+
                           |
                      belongsTo
                           |
+----------+        +-----------+        +----------+
|  Office  |<-------|  Client   |------->|   User   |
+----------+        +-----------+        +----------+
  belongsTo          |         |          belongsTo
                hasMany     belongsToMany
                     |         |
              +-----------+ +------+
              | Property  | | Tag  |
              +-----------+ +------+
```

Keep the diagram focused on direct relationships (1 level deep). For models with many relationships, group by relationship type.

### Step 6: Present Results

Output all findings in a well-structured format with clear section headers. Use code blocks for code snippets and tables for tabular data.

End with a summary:
```
MODEL SUMMARY: Client
======================
Table: clients
Traits: 5 (HasFactory, SoftDeletes, HasActions, HasKeepers, Notifiable)
Fillable fields: 22
Relationships: 12 (4 belongsTo, 5 hasMany, 2 belongsToMany, 1 morphMany)
Scopes: 3
Accessors: 4
Actions: 3
Potential issues: 2
```

## Fix Mode (`fix` or `fix [ModelName]`)

When running in fix mode, automatically apply fixes for detected issues with confirmation before each change:

### Fixable Issues:

1. **Missing $fillable entries**: Add missing columns to the `$fillable` array (excluding id, timestamps, soft deletes, and auto-generated fields).
2. **Missing $casts entries**: Add appropriate casts for date, boolean, JSON, and enum columns.
3. **Missing relationship methods**: Add `belongsTo` methods for FK columns that lack corresponding relationship methods.
4. **Broken relationship references**: Fix incorrect model class references or foreign key names in relationship definitions.
5. **Soft deletes mismatch**: Add `SoftDeletes` trait if migration has `softDeletes()` but model lacks the trait, or vice versa.

**Each fix requires explicit user confirmation before applying.**

## Dry Run Mode (`fix --dry-run`)

Show all the fixes that would be applied (same as fix mode) but do not write any changes. Display the before/after diff for each proposed change.
