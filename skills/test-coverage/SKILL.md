---
name: lc:test-coverage
description: Analyze which models, controllers, and actions have tests and which don't.
argument-hint: "[analyze | directory]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Test Coverage Analysis

Analyze which application classes have corresponding tests and which are untested. Provides a coverage map without requiring actual test execution.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Analyze the full project for test coverage gaps. Read-only report. |
| `[directory]` | Analyze a specific directory (e.g., `app/Models`, `app/Actions`). |

This is an analysis-only skill. It does not modify files. Use `lc:generate-test` to create missing tests.

## Process

### Step 1: Scan Application Classes

Use `Glob` to find all testable classes:

1. **Models**: `app/Models/*.php`
2. **Controllers**: `app/Http/Controllers/**/*.php`
3. **Actions**: `app/Actions/**/*.php`
4. **Services**: `app/Services/*.php`
5. **Jobs**: `app/Jobs/*.php`
6. **Observers**: `app/Observers/*.php`
7. **Livewire/Volt components**: `resources/views/livewire/**/*.blade.php` (only those with PHP class blocks)

If a `[directory]` argument is provided, limit scanning to that directory.

### Step 2: Scan Test Files

Use `Glob` to find all test files:

1. **Feature tests**: `tests/Feature/**/*.php`
2. **Unit tests**: `tests/Unit/**/*.php`

### Step 3: Match Classes to Tests

For each application class, check if a corresponding test exists:

1. **By naming convention**: `App\Models\Property` -> `tests/Feature/Models/PropertyTest.php` or `tests/Unit/Models/PropertyTest.php`
2. **By class reference**: Use `Grep` to search test files for references to the class (e.g., `use App\Models\Property`, `Property::factory()`, `Property::create(`)
3. **By method reference**: Search test files for method names from the class being tested

Classify each class as:
- **TESTED**: A dedicated test file exists with meaningful test methods
- **PARTIALLY_TESTED**: The class is referenced in tests but has no dedicated test file (tested indirectly)
- **UNTESTED**: No test file and no references in any test

### Step 4: Estimate Complexity

For untested classes, estimate testing priority based on:

1. **Number of public methods**: More methods = more to test = higher priority
2. **Database interactions**: Classes that create/update/delete data are higher priority
3. **External service calls**: Classes making API calls or sending emails
4. **Business logic density**: Actions and services with complex logic
5. **User-facing**: Controllers and Livewire components that users interact with directly

Score each untested class:
- **High priority**: Database mutations, user-facing, business logic, external services
- **Medium priority**: Read-only queries, internal helpers, observers
- **Low priority**: Simple data containers, enums, simple accessors

### Step 5: Report

```
TEST COVERAGE ANALYSIS
=======================

SUMMARY
--------
Total application classes:  85
With dedicated tests:       23 (27%)
Partially tested:           12 (14%)
Untested:                   50 (59%)

BY CATEGORY
------------
Category         | Total | Tested | Partial | Untested | Coverage
-----------------|-------|--------|---------|----------|--------
Models           | 18    | 5      | 8       | 5        | 72%
Controllers      | 22    | 8      | 2       | 12       | 45%
Actions          | 15    | 3      | 1       | 11       | 27%
Services         | 8     | 2      | 0       | 6        | 25%
Jobs             | 12    | 3      | 0       | 9        | 25%
Observers        | 5     | 1      | 1       | 3        | 40%
Volt Components  | 5     | 1      | 0       | 4        | 20%

HIGH PRIORITY UNTESTED CLASSES
-------------------------------
These classes have the most impact and should be tested first:

1. App\Http\Controllers\PropertyController (8 public methods, DB mutations)
2. App\Actions\Property\ValidateToPublish (complex business logic)
3. App\Services\IdealistaService (external API integration)
4. App\Jobs\SyncPropertyToFotocasa (external service, queued)
5. App\Http\Controllers\OrganizationStripeWebhookController (payment handling)

FULLY TESTED CLASSES
---------------------
App\Models\User                      -> tests/Unit/Models/UserTest.php (12 tests)
App\Http\Controllers\AuthController  -> tests/Feature/AuthTest.php (8 tests)
...

UNTESTED CLASSES (by category)
-------------------------------
Models:
  - App\Models\Showcase (5 public methods)
  - App\Models\ShowcaseLogger (3 public methods)
  ...

Controllers:
  - App\Http\Controllers\DashboardController (4 methods)
  ...
```

### Step 6: Recommendations

End with actionable recommendations:

```
RECOMMENDATIONS
================
1. Start with high-priority untested classes (listed above)
2. Use lc:generate-test to scaffold test files quickly
3. Focus on controllers with validation rules -- they catch the most bugs
4. Actions with business logic are the most valuable unit tests
5. Consider adding tests before refactoring any existing code
```

## Notes

- This is static analysis. It counts test file existence and class references, not actual code coverage.
- For actual line-by-line coverage, run `php artisan test --coverage` (requires Xdebug or PCOV).
- Indirect testing (a class tested through another test) is common and valid, but dedicated tests are preferred for isolation.
- Volt components with only template code (no PHP class block) are excluded from the analysis.
