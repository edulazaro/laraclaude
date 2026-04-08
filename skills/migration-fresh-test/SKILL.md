---
name: laraclaude:migration-fresh-test
description: Run migrate:fresh in Docker and report errors with fix suggestions.
argument-hint: "[--seed]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Migration Fresh Test

Run `php artisan migrate:fresh` inside the project's Docker container and provide detailed error analysis with fix suggestions when migrations fail.

## Process

### Step 1: Detect Docker Container

1. Use `Read` to examine `docker-compose.yml` in the project root.
2. Identify the PHP/application service (typically named `app`, `laravel`, or similar -- look for the service that mounts the application code and runs PHP).
3. Determine the container name. Convention is usually `{project}_app` or defined via `container_name` in docker-compose.yml.
4. Verify the container is running with `docker ps --filter name={container_name} --format '{{.Names}}'` via `Bash`.
5. If the container is not running, inform the user and suggest `docker compose up -d`.

### Step 2: Pre-flight Checks

1. **Check for schema dump files**: Use `Glob` to look for `database/schema/*.sql`. If found, warn the user that schema dumps can cause migrations to be skipped (old migrations won't run, data imports may be lost). Ask if they want to remove the schema dump first.
2. **Syntax check all migrations**: Run `php -l` on each migration file to catch parse errors before running the full migration. Use `Bash`:
   ```
   docker exec {container} find /var/www/html/database/migrations -name "*.php" -exec php -l {} \;
   ```
   If any syntax errors are found, report them immediately with file path and line number. Read the file to show the problematic code and suggest a fix. Stop here until syntax errors are resolved.

### Step 3: Run migrate:fresh

1. Build the command:
   - Base: `docker exec {container} php artisan migrate:fresh`
   - If `--seed` argument was provided, append `--seed`
   - Always add `--force` to bypass production checks
   - Add `2>&1` to capture both stdout and stderr
2. Run via `Bash` with a generous timeout (migrations with data imports can take minutes).
3. Capture the full output.

### Step 4: Analyze Results

#### If successful:
Report success with:
- Total migrations run
- Time taken
- If `--seed` was used, seeder results
- Any warnings in the output (deprecation notices, etc.)

#### If failed:
Parse the error output to identify:

1. **The failing migration**: Extract the migration filename from the error output (typically shown as "Migrating: {filename}").
2. **The SQL error**: Extract the actual SQL error message.
3. **The error type**: Classify into one of the known patterns below.

### Step 5: Diagnose and Suggest Fixes

For each error, use `Read` to examine the failing migration and related files, then provide a diagnosis:

#### Common Error Patterns

**Foreign Key Constraint Failure**
- Pattern: `Cannot add foreign key constraint` or `SQLSTATE[HY000]: General error: 1215`
- Cause: Referenced table doesn't exist yet (migration order issue) or column types don't match
- Diagnosis: Read the migration, find the FK definition, check if the referenced table's migration has an earlier timestamp
- Fix: Suggest renaming the migration file to adjust timestamp order, or splitting the FK into a separate later migration

**Table Already Exists**
- Pattern: `Table '{table}' already exists` or `SQLSTATE[42S01]`
- Cause: Duplicate CREATE migration, or schema dump + migration both create the same table
- Diagnosis: Search for all migrations that create this table
- Fix: Remove duplicate migration or schema dump

**Column Already Exists**
- Pattern: `Duplicate column name '{column}'` or `SQLSTATE[42S21]`
- Cause: Column added in CREATE migration and again in ALTER migration
- Diagnosis: Find all migrations that add this column
- Fix: Remove the duplicate column addition

**Unknown Column**
- Pattern: `Unknown column '{column}'` or `SQLSTATE[42S22]`
- Cause: Migration references a column that hasn't been created yet, or was dropped
- Diagnosis: Trace the column through all migrations
- Fix: Adjust migration order or fix column name

**Data Type Mismatch**
- Pattern: `SQLSTATE[HY000]: General error: 3780` (referencing column type mismatch)
- Cause: FK column type doesn't match the referenced column (e.g., unsigned vs signed, bigint vs int)
- Diagnosis: Compare column definitions in both tables
- Fix: Ensure both columns use the same type (typically `unsignedBigInteger` or `foreignId`)

**SQL Import Error**
- Pattern: Errors during `DB::unprepared()` with large SQL files
- Cause: SQL syntax issues, character encoding, or memory limits
- Diagnosis: Check the SQL file being imported
- Fix: Suggest splitting the import or increasing PHP memory limit

**Class Not Found**
- Pattern: `Class '{class}' not found`
- Cause: Migration references a model or class that was moved/renamed
- Diagnosis: Search for the class in the codebase
- Fix: Update the import/reference

### Step 6: Provide Fix and Offer to Apply

For each error:

1. Show the error clearly with the migration file path and line number.
2. Explain the root cause.
3. Show the specific code that needs to change.
4. Ask if the user wants to apply the fix.
5. If yes, use `Edit` to apply the fix.
6. After fixing, offer to re-run `migrate:fresh` to verify.

## Output Format

```
PRE-FLIGHT CHECKS
==================
Schema dump: Not found (OK)
Syntax check: 85 migrations checked, 0 errors

MIGRATION RESULTS
==================
Status: FAILED
Failed at: 2025_05_14_214422_create_clients_table.php
Migrations completed before failure: 23/85

ERROR ANALYSIS
===============
Error: Cannot add foreign key constraint
SQL: ALTER TABLE `clients` ADD CONSTRAINT `clients_file_id_foreign` FOREIGN KEY (`file_id`) REFERENCES `files` (`id`)

Root Cause: The `files` table migration (2025_06_01_...) runs AFTER the `clients` table migration (2025_05_14_...).

SUGGESTED FIX
===============
Rename the files migration to run before clients:
  From: 2025_06_01_000000_create_files_table.php
  To:   2025_05_13_000000_create_files_table.php

Apply this fix? [Waiting for confirmation]
```

## Important Notes

- This skill WILL drop all tables and recreate them. It should only be used in development/testing environments.
- Always warn the user that this will destroy all data in the database before proceeding.
- If the project uses SQLite for testing, offer to test against SQLite first as a faster alternative.
