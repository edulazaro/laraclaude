---
name: laraclaude:consolidate-migrations
description: Analyze and consolidate fragmented Laravel migration files. Groups by table, detects dependencies, proposes safe merges.
argument-hint: "[analyze | all | table-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Consolidate Laravel Migrations

Analyze and consolidate fragmented migration files for Laravel projects. Multiple migrations that modify the same table over time can be merged into a single clean migration.

## Modes

The skill operates in three modes based on the argument provided:

### Mode: `analyze` (or no argument)

Perform a read-only analysis of all migration files.

**Steps:**

1. **Scan all migrations** using `Glob` to find `database/migrations/*.php` files.
2. **Parse each migration** using `Read` to extract:
   - `Schema::create('table_name', ...)` -- table creation
   - `Schema::table('table_name', ...)` -- table modification
   - `Schema::dropIfExists('table_name')` -- table drop
   - `Schema::rename('old', 'new')` -- table rename
   - `DB::statement(...)` or `DB::table(...)->update(...)` -- data migrations
3. **Group migrations by table name**. For each table, collect:
   - The original CREATE migration (timestamp, filename, columns defined)
   - All subsequent ALTER migrations (what they add, modify, drop)
   - Any data migrations that touch the table
4. **Detect foreign key dependencies** between tables by parsing `->foreign(...)`, `->foreignId(...)`, `->constrained(...)`, and `->references(...)->on(...)` calls.
5. **Classify each table** into one of these categories:
   - **ALREADY_OPTIMAL**: Only one migration exists for this table, nothing to consolidate
   - **SAFE_TO_MERGE**: Multiple migrations exist, no data migrations, no complex renames. Can be safely consolidated.
   - **MERGE_WITH_CAUTION**: Has data migrations or complex operations that need manual review after merge
   - **DO_NOT_MERGE**: Has rename operations, cross-table data migrations, or other operations that cannot be safely consolidated
6. **Display a summary table** in this format:

```
Table                  | Migrations | Category          | Notes
-----------------------|------------|-------------------|---------------------------
users                  | 5          | SAFE_TO_MERGE     | 4 ALTER migrations
properties             | 8          | MERGE_WITH_CAUTION| 2 data migrations found
locations              | 2          | DO_NOT_MERGE      | SQL data import in migration
clients                | 1          | ALREADY_OPTIMAL   |
```

7. **For each non-optimal table**, list the specific migrations that would be affected.

### Mode: `all`

Consolidate all tables classified as SAFE_TO_MERGE.

**Steps:**

1. Run the `analyze` mode first to identify candidates.
2. Present the list of tables that will be consolidated and **ask for explicit confirmation** before proceeding.
3. For each SAFE_TO_MERGE table, perform the consolidation (see Consolidation Process below).
4. After all consolidations, run `php -l` on every new migration file to validate syntax.
5. Report what was done: files created, files that can be deleted, total reduction.

### Mode: `[table-name]`

Consolidate migrations for a specific table.

**Steps:**

1. Find all migrations that touch the specified table.
2. Classify the table (using the same logic as analyze mode).
3. If SAFE_TO_MERGE: proceed with consolidation after confirmation.
4. If MERGE_WITH_CAUTION: show warnings about data migrations, ask if user wants to proceed. If yes, keep data migrations as separate files.
5. If DO_NOT_MERGE: explain why and stop.
6. If ALREADY_OPTIMAL: inform user and stop.

## Consolidation Process

When consolidating migrations for a table:

1. **Start with the original CREATE migration** as the base.
2. **Apply all ALTER migrations in chronological order** to build the final column state:
   - `$table->string('name')` in CREATE + `$table->string('name', 500)->change()` in ALTER = `$table->string('name', 500)` in final
   - `$table->string('temp')` in CREATE + `$table->dropColumn('temp')` in ALTER = column removed from final
   - New columns added in ALTER migrations are added to the CREATE in the appropriate position
3. **Merge all index definitions** into the CREATE migration.
4. **Merge all foreign key definitions** into the CREATE migration, respecting the dependency order.
5. **Keep the original CREATE migration timestamp** so migration order is preserved.
6. **Extract data migrations** into separate files that run after the consolidated CREATE. These keep their original timestamps.
7. **Write the consolidated migration** using `Write`, replacing the original CREATE file.
8. **List the migrations that are now redundant** (the ALTER-only migrations that were merged) and ask the user to confirm deletion.
9. **Validate syntax** by running `php -l` on the consolidated file via `Bash`.

## Safety Rules

- **NEVER consolidate if this is a production database**. Ask the user to confirm this is a development/staging environment.
- **ALWAYS ask for confirmation** before deleting any migration files.
- **NEVER delete data migrations**. Keep them as separate files even when consolidating schema changes.
- **Preserve the `down()` method** logic. The consolidated `down()` should drop the table entirely.
- **Handle nullable/default changes correctly**: if a column was created as nullable and later changed to NOT NULL with a default, the final state should reflect the NOT NULL with default.
- **Check for schema dump files** (`database/schema/*.sql`). If one exists, warn the user that consolidation may conflict with the schema dump.

## Docker Awareness

If a `docker-compose.yml` exists in the project root, use `docker exec {container_name} php -l {file}` for PHP syntax validation. Detect the container name from the docker-compose.yml service configuration (look for the PHP/app service).

## Output Format

Always provide clear, structured output. Use tables for summaries. Show file paths as absolute paths. When showing migration content, use syntax-highlighted PHP code blocks.
