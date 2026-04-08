---
name: lc:slow-queries
description: Detect queries without indexes, queries in loops, and unbounded selects.
argument-hint: "[analyze | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Slow Query Detection

Detect potential slow queries through static analysis: missing indexes, queries inside loops, unbounded selects, and foreign key columns without indexes.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Scan the entire project for slow query patterns. Read-only report. |
| `fix` | Generate migration(s) to add missing indexes. Asks for confirmation before each change. |
| `fix --dry-run` | Show what index migrations would be generated without writing anything. |
| `[file-path]` | Analyze a specific file for slow query patterns. Read-only. |
| `fix [file-path]` | Fix slow query issues originating from a specific file, with confirmation. |

## Detection Rules

### 1. Missing Indexes on Queried Columns

**What to detect:**
Columns used in `where()`, `whereHas()`, `orderBy()`, `groupBy()`, and `having()` clauses that lack a corresponding index in migrations.

**How to scan:**
1. Use `Grep` to find all query builder calls with column references:
   - `->where('column_name',` and `->where(['column_name' =>`
   - `->whereHas('relation',` (check the closure for column references)
   - `->orderBy('column_name')`
   - `->groupBy('column_name')`
   - `->having('column_name',`
2. For each column found, check migrations for:
   - `$table->index('column_name')` or `$table->index(['column_name', ...])`
   - `$table->unique('column_name')` (unique indexes also serve as indexes)
   - Column defined with `->primary()`, `->unique()`, or `->index()`
   - `$table->foreignId('column_name')` -- some databases auto-index FK columns, some don't
3. Flag columns used in WHERE/ORDER without any index.

**Severity:** Medium (depends on table size)

**Fix:** Generate a migration to add the missing index.

### 2. Queries Inside Loops

**What to detect:**
Database queries executed inside `foreach`, `for`, `while`, `array_map`, `collect()->each()`, or similar iteration constructs.

**How to scan:**
1. Use `Grep` to find loop constructs in PHP files.
2. Use `Read` to load each file with loops and check for query builder or Eloquent calls inside the loop body:
   - `Model::where(`, `Model::find(`, `Model::create(`
   - `DB::table(`, `DB::select(`
   - `->save()`, `->update()`, `->delete()`
   - `->first()`, `->get()`, `->count()`
3. Distinguish between:
   - **N+1 via query**: Direct query calls inside loops (this skill's scope)
   - **N+1 via relationship**: Lazy-loaded relationships (handled by `lc:find-n-plus-one`)

**Severity:** High (multiplied by loop iteration count)

**Fix:** Suggest refactoring to bulk operations:
- `Model::find($id)` in loop -> `Model::whereIn('id', $ids)->get()->keyBy('id')`
- `->save()` in loop -> `Model::upsert($records, ...)`
- `->create()` in loop -> `Model::insert($records)`

### 3. Unbounded Selects

**What to detect:**
Queries using `->get()` without `->limit()`, `->take()`, or `->paginate()` on tables that could grow large.

**How to scan:**
1. Use `Grep` to find all `->get()` calls (excluding `->get('key')` which is Collection::get).
2. Trace back the query chain to check for:
   - `->limit()` or `->take()` present -> safe
   - `->paginate()` or `->simplePaginate()` present -> safe
   - `->where()` with a highly selective condition (e.g., `where('id', $id)`) -> safe
   - None of the above -> potentially unbounded
3. Cross-reference with table size hints:
   - Tables with `timestamps()` and many migration modifications are likely large
   - Pivot tables and log/activity tables grow rapidly

**Severity:** Medium (could cause memory issues on large tables)

**Fix:** Suggest adding `->limit()` or converting to `->paginate()`.

### 4. Missing Indexes on Foreign Key Columns

**What to detect:**
Foreign key columns defined in migrations without explicit indexes. While some databases (MySQL InnoDB) auto-create indexes for FK constraints, columns used as FKs via convention (ending in `_id`) but without actual FK constraints will not have indexes.

**How to scan:**
1. Parse all migrations for columns ending in `_id`.
2. Check if each `_id` column has:
   - A `->foreign()` constraint (MySQL auto-indexes these)
   - An explicit `->index()` definition
   - A `->foreignId()` call (creates both column and index via constraint)
3. Flag `_id` columns that have neither a FK constraint nor an explicit index.

**Severity:** Medium

**Fix:** Generate a migration to add indexes on unindexed FK columns.

### 5. SELECT * Patterns

**What to detect:**
Queries that select all columns when only a few are needed, especially on wide tables.

**How to scan:**
1. Use `Grep` to find queries without `->select()` on tables with many columns (15+).
2. Check for `Model::all()` usage.
3. Check for `->get()` without a preceding `->select()`.

**Severity:** Low (performance impact varies by table width)

**Fix:** Suggest adding `->select()` with only needed columns, especially in list/index views.

## Report Format

```
SLOW QUERY ANALYSIS
====================

CRITICAL (queries in loops)
----------------------------
1. Query inside foreach loop
   File: app/Services/PropertyService.php:78
   Loop: foreach ($propertyIds as $id)
   Query: Property::find($id)
   Impact: ~100 queries per request (estimated from context)
   Fix: Refactor to Property::whereIn('id', $propertyIds)->get()

HIGH (missing indexes on frequently queried columns)
-----------------------------------------------------
1. Missing index on 'properties.status'
   Queried in: 4 files
   - app/Http/Controllers/PropertyController.php:32
   - resources/views/livewire/properties/index.blade.php:15
   - app/Services/SearchService.php:44
   - app/Jobs/SyncPropertyJob.php:22
   Table estimated size: Large (15+ migrations)
   Fix: Add index migration

MEDIUM (unbounded selects)
---------------------------
1. Unbounded ->get() on 'activities' table
   File: app/Http/Controllers/DashboardController.php:18
   Query: Activity::where('organization_id', $id)->get()
   Fix: Add ->paginate(50) or ->limit(100)

SUMMARY
========
Issues found: 12
  Critical (queries in loops):     3
  High (missing indexes):          5
  Medium (unbounded selects):      3
  Low (SELECT * patterns):         1
```

## Fix Mode

In fix mode, generate migrations for missing indexes:

1. Group all missing indexes by table.
2. Generate one migration per table:
   ```php
   Schema::table('properties', function (Blueprint $table) {
       $table->index('status');
       $table->index('organization_id');
   });
   ```
3. Write the migration file with appropriate timestamp.
4. Validate syntax.

For queries-in-loops and unbounded selects, show the suggested refactoring code but do not auto-apply (risk of breaking logic). Present as recommendations with file path and line number.

## Notes

- This is static analysis. Actual query performance depends on data volume, database configuration, and query plans.
- Use `EXPLAIN` on actual queries for definitive performance analysis.
- Some "unbounded" queries may be intentionally loading small datasets. Review context before adding limits.
- Foreign key columns created with `$table->foreignId()` automatically get indexes in MySQL via the FK constraint. These are safe.
- Composite indexes may be more effective than single-column indexes for queries with multiple WHERE conditions. Note this in recommendations.
