---
name: laraclaude:unused-columns
description: Detect database columns that are never referenced in the codebase.
argument-hint: "[table-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Detect Unused Columns

Find database columns that exist in migrations (or the live database) but are never referenced anywhere in the application codebase. These are candidates for removal.

## Process

### Step 1: Get Column List

**If a `[table-name]` argument is provided**, analyze only that table. **If no argument**, analyze all tables.

#### Method A: From Migrations (preferred, no DB needed)

1. Use `Glob` to find all `database/migrations/*.php` files.
2. For each table, parse CREATE and ALTER migrations to build the final column list:
   - `$table->string('name')` -- column: name
   - `$table->foreignId('user_id')` -- column: user_id
   - `$table->timestamps()` -- columns: created_at, updated_at
   - `$table->softDeletes()` -- column: deleted_at
   - `$table->morphs('taggable')` -- columns: taggable_type, taggable_id
   - `$table->rememberToken()` -- column: remember_token
   - `$table->dropColumn('name')` -- remove column from list
3. Track column renames: `$table->renameColumn('old', 'new')`

#### Method B: From Database (if Docker is available)

```
docker exec {container} php artisan tinker --execute="
    \$columns = Schema::getColumnListing('{table}');
    echo implode(PHP_EOL, \$columns);
"
```

### Step 2: Exclude Standard Columns

The following columns are considered "always used" and should be excluded from the analysis:

- `id` -- primary key
- `created_at`, `updated_at` -- timestamps
- `deleted_at` -- soft deletes
- `slug` -- commonly used for URLs
- `remember_token` -- authentication
- `email_verified_at` -- email verification
- `password` -- authentication

These are still tracked but reported separately as "Standard columns (excluded from analysis)".

### Step 3: Search for Column References

For each non-standard column, search the entire codebase for references using `Grep`. Search in these locations and patterns:

#### A. Model files (`app/Models/*.php`)
- `$fillable` array entries
- `$casts` array entries
- `$hidden` array entries
- `$appends` array entries
- Accessor/mutator definitions (`get{Column}Attribute`, `set{Column}Attribute`, `Attribute::make`)
- Relationship definitions (foreign key parameters)
- Scope definitions (`scope` methods that reference the column)
- `$table->` references in model methods

#### B. Blade views (`resources/views/**/*.blade.php`)
- `$model->column_name` access
- `wire:model="column_name"` bindings
- `wire:model.live="column_name"` bindings
- `{{ $column_name }}` direct output
- `@if($model->column_name)` conditions

#### C. Controllers and Actions (`app/Http/Controllers/**/*.php`, `app/Actions/**/*.php`)
- `$request->column_name` or `$request->input('column_name')`
- `->where('column_name', ...)`
- `->orderBy('column_name')`
- `->select('column_name')`
- `->pluck('column_name')`
- `->groupBy('column_name')`
- Array keys: `'column_name' => $value`

#### D. Observers (`app/Observers/*.php`)
- `$model->column_name` access
- `$model->isDirty('column_name')`
- `$model->wasChanged('column_name')`

#### E. Jobs and Services (`app/Jobs/*.php`, `app/Services/*.php`)
- Any reference to the column name

#### F. Migrations (`database/migrations/*.php`)
- References beyond the column definition itself (e.g., data migrations using the column)

#### G. Seeders and Factories (`database/seeders/*.php`, `database/factories/*.php`)
- Column assignments in factory definitions or seeders

#### H. Config and Routes
- Column references in config files or route definitions

### Step 4: Handle False Positives

Column names that are common English words may match in unrelated contexts. Apply these filters:

1. **Require context**: The match should be near a model reference, query builder call, or Blade variable access. A standalone word match in a comment or text string is not a reference.
2. **Check for dynamic access**: Some columns may be accessed dynamically via `$model->$field` or `$model->getAttribute($field)`. These are harder to detect statically -- flag columns that appear in arrays that might be iterated for dynamic access.
3. **Check for JSON/array column access**: Columns of type JSON might be accessed via `->column->nested_key` or `->column['key']`.

### Step 5: Classify Results

For each column, classify into:

- **UNUSED**: Zero references found anywhere in the codebase. Strong candidate for removal.
- **MIGRATION_ONLY**: Referenced only in migrations (definition and possibly data migrations). Likely unused by application code.
- **POSSIBLY_UNUSED**: Only 1-2 references found, and those references may be dynamic or ambiguous. Needs manual review.
- **IN_USE**: Multiple clear references across the codebase. Not a candidate for removal.

### Step 6: Report Findings

For a specific table:

```
UNUSED COLUMNS ANALYSIS: properties
=====================================

Columns analyzed: 42 (excluding 5 standard columns)

UNUSED (0 references):
  - legacy_portal_id      | bigInteger, nullable | No references found
  - old_reference_code     | string(50), nullable | No references found
  - temp_import_flag       | boolean, default:false | No references found

MIGRATION_ONLY (referenced only in migrations):
  - migration_batch_id     | integer, nullable | Found in: 1 migration data update

POSSIBLY_UNUSED (needs review):
  - internal_notes         | text, nullable | Found in: 1 factory definition only

IN USE (45+ columns):
  - name, address, price, ... (not listed individually)

STANDARD (excluded):
  - id, created_at, updated_at, deleted_at, slug
```

For all tables (no argument):

```
UNUSED COLUMNS SUMMARY
========================

Table                | Total Cols | Unused | Migration Only | Possibly Unused
---------------------|------------|--------|----------------|----------------
properties           | 47         | 3      | 1              | 1
clients              | 28         | 0      | 0              | 2
operations           | 12         | 1      | 0              | 0
schedules            | 15         | 0      | 0              | 0
...

TOTAL UNUSED COLUMNS: 8 across 4 tables
```

Then list all unused columns with details.

### Step 7: Suggest Removal

For each UNUSED column, suggest a migration to remove it:

```php
// Suggested migration to remove unused columns from 'properties' table
Schema::table('properties', function (Blueprint $table) {
    $table->dropColumn(['legacy_portal_id', 'old_reference_code', 'temp_import_flag']);
});
```

Offer to generate the migration file. If the user agrees, create it with an appropriate timestamp.

## Important Caveats

- **Dynamic access**: Columns accessed via `$model->getAttribute($dynamicField)` or similar dynamic patterns cannot be detected statically. Note this in the report.
- **API responses**: Columns may be used in API responses via `toArray()` or resources. Check for `$hidden` and `$visible` model properties.
- **Package usage**: Some columns may be used by installed packages (e.g., Spatie packages, Laravel Scout). Check `vendor/` if unsure.
- **Third-party integrations**: Columns like `google_id`, `stripe_id` may be used by external service integrations. Cross-check with service configuration.
- **Column name collisions**: Short column names like `type`, `name`, `status` will have many grep matches. Use context-aware matching to avoid false positives -- search for `'{column_name}'` (quoted) and `->column_name` patterns rather than just the bare word.
- **Do NOT search for bare column names** that are common words (type, name, status, active, code, etc.). Instead, search for them in specific contexts: `'column_name'` in arrays, `->column_name` for property access, `"column_name"` in query strings.
