---
name: lc:dead-code
description: Find unused classes, methods, routes, views, and imports.
argument-hint: "[analyze | clean | clean --dry-run | directory]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Dead Code Detection

Find unused classes, methods, routes, Blade views, and import statements through static analysis.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Scan the entire project for dead code. Read-only report. |
| `clean` | Remove confirmed dead code. Asks for confirmation before each removal. |
| `clean --dry-run` | Show what would be removed without deleting anything. |
| `[directory]` | Analyze a specific directory for dead code. Read-only. |

## Detection Rules

### 1. Unused Classes

**What to detect:**
Classes in `app/` that are never instantiated, referenced, imported, or used anywhere else in the codebase.

**How to scan:**
1. Use `Glob` to find all PHP classes in `app/` (excluding vendor).
2. For each class, extract the fully qualified class name.
3. Use `Grep` to search the entire codebase for:
   - `use App\ClassName;` import statements
   - `ClassName::` static calls
   - `new ClassName(` instantiation
   - `ClassName::class` references (in route files, service providers, model arrays, etc.)
   - Type hints: `function foo(ClassName $param)`
4. Exclude self-references (the class referencing itself).
5. Check special registration locations:
   - `app/Providers/*.php` for service provider registrations
   - `config/*.php` for class references in config
   - `routes/*.php` for controller references
   - `composer.json` for autoload references
   - Model `$actions`, `$casts` arrays for action/enum classes

**Classes to skip (never flag as unused):**
- Models (may be used via Eloquent, route model binding, morph maps)
- Service providers (registered in `config/app.php` or `bootstrap/providers.php`)
- Middleware (registered in kernel or route groups)
- Commands (auto-discovered by Laravel)
- Observers (registered in service providers)
- Policies (auto-discovered by naming convention)

**Severity:** Medium

**Clean action:** Delete the class file after confirmation.

### 2. Unused Public Methods

**What to detect:**
Public methods on classes that are never called from outside the class.

**How to scan:**
1. For each class, extract all public method names.
2. Use `Grep` to search for calls to each method:
   - `->methodName(` instance calls
   - `::methodName(` static calls
   - `'methodName'` string references (for dynamic dispatch, route actions, etc.)
3. Exclude framework-convention methods that are called implicitly:
   - `mount()`, `render()`, `boot()`, `booted()` (Livewire/Eloquent)
   - `handle()` (Jobs, Actions, Commands)
   - `up()`, `down()` (Migrations)
   - `rules()`, `messages()` (Validation)
   - `authorize()` (Form Requests)
   - Relationship methods (called by Eloquent magic)
   - Scope methods (called via `Model::scopeName()`)
   - Accessor/mutator methods

**Severity:** Low (methods may be used dynamically or reserved for future use)

**Clean action:** Remove the method after confirmation. Show the full method body before deletion so the user can review.

### 3. Unused Routes

**What to detect:**
Routes defined in `routes/*.php` that are never linked to via `route()`, `url()`, or `action()` helpers.

**How to scan:**
1. Use `Read` to parse all route files and extract named routes.
2. For each named route, use `Grep` to search for:
   - `route('route.name')` calls
   - `route('route.name',` calls with parameters
   - `Route::has('route.name')` checks
   - Links in Blade: `href="{{ route('route.name') }}"`
   - Redirects: `redirect()->route('route.name')`
3. Also check for unnamed routes by searching for the URI pattern.
4. Exclude API routes that may be called by external clients.

**Severity:** Low (routes may be used by external systems or planned for future use)

**Clean action:** Comment out or remove the route definition after confirmation.

### 4. Unused Blade Views

**What to detect:**
Blade view files in `resources/views/` that are never rendered.

**How to scan:**
1. Use `Glob` to find all `resources/views/**/*.blade.php` files.
2. For each view, derive the view name (e.g., `resources/views/livewire/properties/index.blade.php` -> `livewire.properties.index`).
3. Use `Grep` to search for references:
   - `view('view.name')` or `view("view.name")`
   - `@include('view.name')` or `@include("view.name")`
   - `@extends('view.name')`
   - `@livewire('component.name')`
   - `<livewire:component-name />`
   - `@component('view.name')`
   - `<x-component-name` (for component views)
4. For Livewire Volt components, they are auto-discovered by path -- check if a route or `@livewire` directive references them.

**Views to skip (never flag as unused):**
- Layout files in `resources/views/layouts/` or `resources/views/components/layouts/`
- Component views in `resources/views/components/` (used via `<x-name>`)
- Error pages in `resources/views/errors/`
- Email templates in `resources/views/mail/` or `resources/views/emails/`

**Severity:** Low

**Clean action:** Delete the view file after confirmation.

### 5. Unused Imports (use statements)

**What to detect:**
`use` statements at the top of PHP files that import classes never referenced in the file body.

**How to scan:**
1. For each PHP file, extract all `use` import statements.
2. For each import, extract the class name (last segment of the namespace).
3. Search the file body (excluding the use statement itself) for:
   - The class name as a type hint, return type, or standalone reference
   - The class name in PHPDoc comments (`@param ClassName`)
   - Static calls `ClassName::`
   - Instantiation `new ClassName`
4. Flag imports where the class name is not found in the file body.

**Severity:** Very Low (cosmetic, no runtime impact)

**Clean action:** Remove the unused `use` statement.

## Report Format

```
DEAD CODE ANALYSIS
===================

UNUSED CLASSES (3 found)
-------------------------
1. App\Services\LegacyImportService
   File: app/Services/LegacyImportService.php
   Reason: Zero references in codebase
   Lines: 145

2. App\Actions\Property\OldExportAction
   File: app/Actions/Property/OldExportAction.php
   Reason: Zero references in codebase (not in model $actions)
   Lines: 67

UNUSED ROUTES (2 found)
-------------------------
1. Route: 'admin.legacy.import' (GET /admin/legacy/import)
   File: routes/app.php:142
   Reason: No route() call found

UNUSED VIEWS (1 found)
------------------------
1. View: livewire.admin.legacy-import
   File: resources/views/livewire/admin/legacy-import.blade.php
   Reason: No @livewire or route reference

UNUSED IMPORTS (15 found)
--------------------------
1. app/Http/Controllers/PropertyController.php:8
   use App\Models\Office;  (never referenced)
2. app/Services/SearchService.php:5
   use Illuminate\Support\Str;  (never referenced)
...

SUMMARY
========
Category          | Found | Total Lines
------------------|-------|------------
Unused classes    | 3     | 312
Unused methods    | 7     | 89
Unused routes     | 2     | -
Unused views      | 1     | 45
Unused imports    | 15    | 15
Total dead code   | 28    | ~461 lines
```

## Clean Mode

In clean mode, process each category in order (classes, methods, routes, views, imports):

1. Show the item to be removed with full context.
2. **Ask for confirmation** before each removal.
3. For classes and views: delete the file.
4. For methods: use `Edit` to remove the method from the class.
5. For routes: use `Edit` to remove or comment out the route.
6. For imports: use `Edit` to remove the `use` statement.

## Notes

- **False positives are expected.** Classes may be used via:
  - Dynamic instantiation (`app($className)`)
  - Config references (`config('key')` resolving to a class)
  - Service container bindings
  - Event listeners registered as strings
  - Morph maps in `AppServiceProvider`
- Always err on the side of caution. When in doubt, flag but do not auto-clean.
- Methods on interfaces/contracts are never "unused" even if only called via the interface.
- Test files reference application classes but should not be considered "usage" for dead code purposes (a class only used in tests may still be dead).
- Blade views can be referenced dynamically: `view("livewire.{$type}.{$action}")` -- these are not detectable statically.
