---
name: laraclaude:check-foreign-keys
description: Detect broken, orphaned, or missing foreign key constraints in Laravel migrations and models.
argument-hint: "[table-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Check Foreign Keys

Detect inconsistencies between foreign key constraints defined in migrations and relationships defined in Eloquent models. Finds broken, orphaned, or missing constraints.

## Process

### Step 1: Collect Foreign Keys from Migrations

1. Use `Glob` to find all `database/migrations/*.php` files.
2. Use `Read` and `Grep` to parse each migration for foreign key definitions:
   - `$table->foreign('column')->references('id')->on('table')` -- explicit FK
   - `$table->foreignId('table_id')->constrained()` -- conventional FK
   - `$table->foreignId('column')->constrained('table')` -- explicit constrained FK
   - `$table->foreignIdFor(Model::class)` -- model-based FK
3. For each FK, record:
   - Source table and column
   - Referenced table and column
   - `onDelete` action (cascade, set null, restrict, no action)
   - `onUpdate` action
   - Migration file path and line number

### Step 2: Collect Relationships from Models

1. Use `Glob` to find all `app/Models/*.php` files.
2. Use `Read` to parse each model for:
   - `belongsTo(Model::class, 'foreign_key')` -- implies FK on this model's table
   - `hasMany(Model::class, 'foreign_key')` -- implies FK on the related model's table
   - `hasOne(Model::class, 'foreign_key')` -- implies FK on the related model's table
   - `belongsToMany(Model::class, 'pivot_table')` -- implies FKs on pivot table
   - `morphTo()`, `morphMany()`, `morphOne()` -- polymorphic (no FK expected)
3. For each relationship, record:
   - Model class and file path
   - Relationship type and method name
   - Related model
   - Foreign key column (explicit or conventional)
   - Line number

### Step 3: Cross-Reference and Detect Issues

Check for these categories of problems:

#### A. FK in migration but no relationship in model
A foreign key constraint exists in a migration, but neither the source model nor the target model defines a corresponding relationship.

**Severity:** Warning (the FK works at DB level, but there's no Eloquent relationship to use it)

#### B. Relationship in model but no FK in migration
A `belongsTo` or similar relationship exists in a model, but there's no matching foreign key constraint in any migration.

**Severity:** Error (data integrity is not enforced at DB level)

#### C. FK references non-existent table
A foreign key references a table that is never created in any migration, or is dropped before this FK is created.

**Severity:** Critical (migration will fail)

#### D. FK references non-existent column
A foreign key references a column on the target table that doesn't exist in the target table's CREATE migration.

**Severity:** Critical (migration will fail)

#### E. onDelete action conflicts
- `onDelete('set null')` on a column that is NOT NULL -- will cause runtime errors
- `onDelete('cascade')` on a table with soft deletes -- may bypass soft delete logic
- No `onDelete` specified -- defaults to RESTRICT which may cause unexpected constraint violations

**Severity:** Warning

#### F. Migration order issues
A migration creates a FK referencing a table whose CREATE migration has a later timestamp.

**Severity:** Critical (migrate:fresh will fail)

#### G. Duplicate foreign keys
The same FK is defined in multiple migrations (e.g., created in one, dropped and re-created in another without the drop).

**Severity:** Warning

### Step 4: Report Findings

If a `[table-name]` argument is provided, filter results to only show issues involving that table (as source or target).

Present findings grouped by severity:

```
CRITICAL ISSUES (will cause failures)
======================================

1. FK references non-existent table
   Migration: database/migrations/2025_05_01_create_orders_table.php (line 15)
   FK: orders.customer_id -> customers.id
   Problem: Table 'customers' is not created in any migration
   Fix: Create a migration for the 'customers' table, or fix the reference

WARNING ISSUES (potential problems)
====================================

1. Relationship without FK constraint
   Model: App\Models\Order (line 25)
   Relationship: belongsTo(Customer::class)
   Expected FK: orders.customer_id -> customers.id
   Fix: Add foreign key in migration:
   $table->foreign('customer_id')->references('id')->on('customers')->onDelete('cascade');
```

### Step 5: Summary Table

End with a summary:

```
Category                              | Count
--------------------------------------|------
Foreign keys in migrations            | 45
Relationships in models               | 62
Critical issues                       | 2
Warnings                              | 8
Tables with no issues                 | 30
```

## Special Handling

- **Polymorphic relationships** (`morphTo`, `morphMany`, `morphOne`, `morphToMany`): These intentionally have no FK constraints. Do not report them as missing FKs. Note them separately as "Polymorphic relationships (no FK expected)".
- **Pivot tables** for `belongsToMany`: Check that both FKs exist on the pivot table.
- **Custom FK column names**: When `belongsTo(Model::class, 'custom_column')` uses a non-conventional name, match by the explicit column name.
- **Morph map**: If the project uses `Relation::morphMap()` in `AppServiceProvider`, note this when reporting polymorphic relationships.

## Notes

- This is a static analysis tool. It reads migration files and model files only -- it does not connect to the database.
- For runtime FK checking against an actual database, see `laraclaude:orphaned-records`.
