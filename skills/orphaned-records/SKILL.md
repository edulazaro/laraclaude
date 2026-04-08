---
name: laraclaude:orphaned-records
description: Find orphaned database records where foreign key parent records don't exist.
argument-hint: "[table-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Find Orphaned Records

Detect orphaned database records where a foreign key column references a parent record that no longer exists. This runs actual queries against the database via Docker.

## Process

### Step 1: Detect Docker Container

1. Use `Read` to examine `docker-compose.yml` in the project root.
2. Identify the PHP/application service container name (look for `container_name` or derive from service name).
3. Verify the container is running using `Bash`:
   ```
   docker ps --filter name={container_name} --format '{{.Names}}'
   ```
4. If not running, inform the user and suggest `docker compose up -d`.

### Step 2: Identify Foreign Key Relationships

If a `[table-name]` argument is provided, only analyze that table. Otherwise, analyze all tables.

**Method A: From Migrations (primary)**
1. Use `Glob` to find all `database/migrations/*.php` files.
2. Use `Grep` and `Read` to parse foreign key definitions:
   - `$table->foreign('column')->references('ref_column')->on('ref_table')`
   - `$table->foreignId('ref_table_id')->constrained()`
   - `$table->foreignId('column')->constrained('ref_table')`
   - `$table->foreignIdFor(Model::class)`
3. Build a list of FK relationships: `{table}.{column} -> {ref_table}.{ref_column}`

**Method B: From Database (supplementary)**
If migrations are complex or unclear, query the database directly:
```
docker exec {container} php artisan tinker --execute="
    \$results = DB::select('
        SELECT TABLE_NAME, COLUMN_NAME, REFERENCED_TABLE_NAME, REFERENCED_COLUMN_NAME
        FROM information_schema.KEY_COLUMN_USAGE
        WHERE REFERENCED_TABLE_NAME IS NOT NULL
        AND TABLE_SCHEMA = DB::getDatabaseName()
    ');
    foreach (\$results as \$r) {
        echo \$r->TABLE_NAME . '.' . \$r->COLUMN_NAME . ' -> ' . \$r->REFERENCED_TABLE_NAME . '.' . \$r->REFERENCED_COLUMN_NAME . PHP_EOL;
    }
"
```

### Step 3: Check for Orphaned Records

For each FK relationship, run a query to find records where the parent doesn't exist:

```
docker exec {container} php artisan tinker --execute="
    \$count = DB::table('{table}')
        ->whereNotNull('{column}')
        ->whereNotIn('{column}', DB::table('{ref_table}')->select('{ref_column}'))
        ->count();
    echo '{table}.{column} -> {ref_table}.{ref_column}: ' . \$count . ' orphans';
"
```

For tables with many records, use a LEFT JOIN approach for better performance:
```sql
SELECT COUNT(*) FROM {table} t
LEFT JOIN {ref_table} r ON t.{column} = r.{ref_column}
WHERE t.{column} IS NOT NULL AND r.{ref_column} IS NULL
```

### Step 4: Get Sample Data

For each relationship with orphaned records, fetch sample IDs and values:

```
docker exec {container} php artisan tinker --execute="
    \$samples = DB::table('{table}')
        ->select('id', '{column}')
        ->whereNotNull('{column}')
        ->whereNotIn('{column}', DB::table('{ref_table}')->select('{ref_column}'))
        ->limit(5)
        ->get();
    foreach (\$samples as \$s) {
        echo 'ID: ' . \$s->id . ' | {column}: ' . \$s->{column} . PHP_EOL;
    }
"
```

### Step 5: Check Soft Deletes

Before reporting orphans, check if the referenced table uses soft deletes:

1. Use `Grep` to check if the referenced model uses `SoftDeletes` trait.
2. If it does, also check if the "missing" parent records exist but are soft-deleted:
   ```sql
   SELECT COUNT(*) FROM {table} t
   LEFT JOIN {ref_table} r ON t.{column} = r.{ref_column}
   WHERE t.{column} IS NOT NULL AND r.{ref_column} IS NULL
   ```
   vs.
   ```sql
   SELECT COUNT(*) FROM {table} t
   INNER JOIN {ref_table} r ON t.{column} = r.{ref_column} AND r.deleted_at IS NOT NULL
   WHERE t.{column} IS NOT NULL
   ```
3. Distinguish between truly orphaned records and records pointing to soft-deleted parents.

### Step 6: Report Findings

Present findings in a structured format:

```
ORPHANED RECORDS REPORT
========================

Table: operations
-----------------
Column: client_seller_id -> clients.id
  Orphaned records: 12
  Soft-deleted parents: 3 (these have deleted parent records)
  Truly orphaned: 9 (parent record does not exist at all)
  Sample IDs: 45, 67, 89, 102, 115
  FK onDelete action: SET NULL

Column: property_id -> properties.id
  Orphaned records: 0 (clean)

Table: activities
-----------------
Column: activitable_id (polymorphic)
  Note: Polymorphic relationships cannot be checked via FK constraints.
  Skipped -- use manual inspection if needed.
```

### Step 7: Suggest Fixes

For each table with orphaned records, suggest appropriate fixes based on the FK's onDelete action and business context:

#### Option A: Delete Orphaned Records
```php
// Delete orphaned records from {table} where {column} references non-existent {ref_table} records
DB::table('{table}')
    ->whereNotNull('{column}')
    ->whereNotIn('{column}', DB::table('{ref_table}')->select('{ref_column}'))
    ->delete();
```

#### Option B: Set to NULL (if column is nullable)
```php
// Set orphaned FK values to NULL
DB::table('{table}')
    ->whereNotNull('{column}')
    ->whereNotIn('{column}', DB::table('{ref_table}')->select('{ref_column}'))
    ->update(['{column}' => null]);
```

#### Option C: Create a Migration to Clean Up
Suggest creating a migration that runs the cleanup, so it's tracked and reversible:
```php
public function up()
{
    // Clean orphaned records in {table}
    DB::statement('DELETE FROM {table} WHERE {column} NOT IN (SELECT {ref_column} FROM {ref_table}) AND {column} IS NOT NULL');
}
```

### Step 8: Offer to Execute Fix

Ask the user if they want to:
1. Execute the fix directly via tinker (for development environments)
2. Generate a migration file for the fix (for production-safe approach)
3. Do nothing (just reporting)

If executing directly, always ask for confirmation and show the exact query that will run.

### Summary Table

End with a summary:

```
SUMMARY
========
Tables checked:          35
FK relationships checked: 62
Tables with orphans:     4
Total orphaned records:  47
Soft-deleted parents:    8
Truly orphaned:          39
Polymorphic (skipped):   5
```

## Important Notes

- This skill runs **actual database queries**. It should be used in development/staging environments.
- Polymorphic relationships (`morphTo`, `morphMany`) cannot be checked via FK constraints since they use type+id columns. These are flagged but skipped.
- Large tables may take time to scan. For tables with millions of rows, the LEFT JOIN approach is used for better performance.
- Always check for soft deletes before suggesting deletion of "orphaned" records.
- Never execute destructive queries without explicit user confirmation.
