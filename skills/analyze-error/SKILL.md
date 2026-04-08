---
name: laraclaude:analyze-error
description: Paste a Laravel stacktrace and get root cause analysis with suggested fix.
argument-hint: "(paste error after invoking)"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Analyze Laravel Error

Parse a Laravel stacktrace or error message, identify the root cause, read the relevant source files, and provide a specific fix.

## Process

### Step 1: Receive and Parse the Error

The user will paste an error after invoking this skill. Parse the error output to extract:

1. **Exception class**: e.g., `Illuminate\Database\QueryException`, `ErrorException`, `BadMethodCallException`
2. **Error message**: The human-readable description of what went wrong
3. **File and line**: The exact file and line number where the error occurred
4. **SQL query** (if applicable): The raw SQL that failed
5. **Stack trace**: The chain of method calls leading to the error

If the error is a standard Laravel error page (HTML), extract the relevant information from the HTML structure. If it is a CLI error, parse the text output.

### Step 2: Read the Source File

1. Use `Read` to load the file where the error occurred, focusing on the line number from the stacktrace with surrounding context (approximately 20 lines before and after).
2. Identify the exact code that triggered the error.
3. If the error originated in vendor code, trace back up the stack trace to find the first application file (in `app/`, `routes/`, `resources/`, `database/`, `config/`) -- that is where the fix needs to happen.

### Step 3: Read Related Files

Based on the error type, read additional files for context:

#### For Database/Query Errors:
- The model file involved (from the stack trace or query)
- The migration that defines the table/columns
- The observer (if the error happens during save/create)

#### For Relationship Errors:
- The model defining the relationship
- The related model
- The migration for both tables (to check FK definitions)

#### For View/Blade Errors:
- The Blade view file
- The Livewire component or controller that renders it
- The data passed to the view

#### For Route Errors:
- The routes file
- The controller or Livewire component for the route
- Middleware definitions

#### For Authentication/Authorization Errors:
- The auth configuration (`config/auth.php`)
- The relevant guard and provider setup
- The policy file if applicable

### Step 4: Identify Root Cause

Classify the error into one of these common patterns and provide specific diagnosis:

#### Database Errors

| Pattern | Exception | Root Cause |
|---------|-----------|------------|
| `SQLSTATE[42S02]` | QueryException | Table doesn't exist -- migration not run or wrong table name |
| `SQLSTATE[42S22]` | QueryException | Column doesn't exist -- typo, migration not run, or column removed |
| `SQLSTATE[23000]` | QueryException | Integrity constraint violation -- duplicate entry, null in NOT NULL, FK violation |
| `SQLSTATE[42000]` | QueryException | Syntax error in SQL -- usually from raw queries or incorrect query builder usage |
| `SQLSTATE[HY000]: 1215` | QueryException | Cannot add FK constraint -- type mismatch or referenced table missing |
| `SQLSTATE[22001]` | QueryException | Data too long -- string exceeds column length |

#### Eloquent/Model Errors

| Pattern | Exception | Root Cause |
|---------|-----------|------------|
| `Call to undefined relationship` | BadMethodCallException | Relationship method doesn't exist on model |
| `Property does not exist` | ErrorException | Accessing undefined property (possible missing relationship or accessor) |
| `Mass assignment` | MassAssignmentException | Column not in $fillable or $guarded is blocking it |
| `Model not found` | ModelNotFoundException | `findOrFail()` or `firstOrFail()` with non-existent ID |
| `Call to a member function on null` | ErrorException | Accessing method on null relationship result |

#### View/Blade Errors

| Pattern | Exception | Root Cause |
|---------|-----------|------------|
| `Undefined variable` | ErrorException | Variable not passed from controller/component to view |
| `Property not found on component` | Livewire error | Wire:model bound to non-existent property |
| `Component not found` | ComponentNotFoundException | Livewire component class missing or not registered |
| `View not found` | InvalidArgumentException | Blade file doesn't exist at expected path |

#### Authentication/Authorization

| Pattern | Exception | Root Cause |
|---------|-----------|------------|
| `Unauthenticated` | AuthenticationException | User not logged in, session expired, or wrong guard |
| `This action is unauthorized` | AuthorizationException | Policy check failed |
| `Route [login] not defined` | RouteNotFoundException | Missing named route for auth redirect |

#### Type/Logic Errors

| Pattern | Exception | Root Cause |
|---------|-----------|------------|
| `Typed property must not be accessed before initialization` | Error | Livewire property declared with type but no default value |
| `Cannot access offset of type null` | TypeError | Trying to array-access a null value |
| `Argument #N must be of type X, Y given` | TypeError | Wrong argument type passed to method |
| `Enum case not found` | ValueError | Invalid value for PHP enum cast |

### Step 5: Provide the Fix

For each error, provide:

1. **Root cause explanation** (1-2 sentences explaining WHY this happened, not just WHAT happened)
2. **The specific file and line that needs to change**
3. **The exact code fix** showing before and after:
   ```php
   // BEFORE (line 45 in app/Models/Property.php):
   return $this->office->name;

   // AFTER:
   return $this->office?->name;
   ```
4. **Prevention advice** (how to avoid this in the future)

### Step 6: Check for Related Issues

After identifying the primary fix, check if the same pattern exists elsewhere:

1. Use `Grep` to search for similar problematic patterns in other files.
2. If found, report them as "Related issues that may need the same fix" with file paths and line numbers.

### Step 7: Offer to Apply

Ask the user if they want to apply the fix. If yes, use `Edit` to make the change.

## Error Parsing Tips

### Laravel CLI Errors
```
   Illuminate\Database\QueryException

  SQLSTATE[42S22]: Column not found: 1054 Unknown column 'properties.group_id' in 'where clause'
  (Connection: mysql, SQL: select * from `properties` where `properties`.`group_id` = 1)

  at vendor/laravel/framework/src/Illuminate/Database/Connection.php:829
  ...
  at app/Http/Controllers/PropertyController.php:25
```
Extract: Exception class, SQL state, column name, table, and the app file from the trace.

### Laravel Log Errors
```
[2026-04-06 10:15:23] production.ERROR: Call to a member function format() on null
{"exception":"[object] (Error(code: 0): ... at /var/www/html/app/Models/Schedule.php:45)"}
```
Extract from JSON structure within the log line.

### Livewire Errors
```
Livewire\Exceptions\PropertyNotFoundException
Property [$selectedUser] not found on component: [calendar]
```
These reference the Livewire component name, which maps to a Blade file in `resources/views/livewire/`.

### Vite/Asset Errors
If the error is related to Vite, assets, or frontend compilation, check:
- `vite.config.js` configuration
- `package.json` dependencies
- Whether `npm run dev` is running

## Known Project-Specific Issues

Based on the project's CLAUDE.md, watch for these specific patterns:

1. **Schema dump conflicts**: If migrate:fresh errors mention missing tables, check for `database/schema/mysql-schema.sql`.
2. **Foreign key order**: The project has known FK dependency issues (files -> clients -> properties).
3. **Organization rename**: Old code may still reference `group_id` instead of `organization_id`, `Group` instead of `Organization`.
4. **Custom notifications**: Code using `DatabaseNotification` should use `App\Models\Notification` instead.
5. **Translation errors**: Using `__()` instead of `text()` will not work with Laratext.

## Output Format

```
ERROR ANALYSIS
===============

Exception: Illuminate\Database\QueryException
Message: SQLSTATE[42S22]: Column not found: 1054 Unknown column 'properties.group_id'

ROOT CAUSE
===========
The 'properties' table was renamed from 'group_id' to 'organization_id' in the
Organization rename migration (2025_12_25_023229), but this query still uses the
old column name.

LOCATION
=========
File: app/Http/Controllers/PropertyController.php
Line: 25
Code: Property::where('group_id', $organizationId)->get();

FIX
====
Change 'group_id' to 'organization_id':

  // Line 25
  - Property::where('group_id', $organizationId)->get();
  + Property::where('organization_id', $organizationId)->get();

RELATED ISSUES
===============
Found 3 other files with the same pattern:
  - app/Services/PropertyService.php:42
  - app/Jobs/SyncPropertyJob.php:18
  - resources/views/livewire/properties/index.blade.php:33

Apply fix? [Waiting for confirmation]
```
