---
name: lc:find-n-plus-one
description: Detect N+1 query problems in Blade views, Livewire components, and controllers.
argument-hint: "[analyze | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Find N+1 Query Problems

Statically analyze Blade views, Livewire/Volt components, and controllers to detect N+1 query problems where relationships are accessed inside loops without eager loading.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Scan the entire project for N+1 problems. Read-only report. |
| `fix` | Scan and auto-add `with()` eager loading to queries. Asks for confirmation before each change. |
| `fix --dry-run` | Show what `with()` calls would be added without changing anything. |
| `[file-path]` | Analyze a specific file and trace its data sources. Read-only. |
| `fix [file-path]` | Fix N+1 issues in a specific file, with confirmation. |

## What is N+1?

An N+1 query problem occurs when code accesses a relationship on a model inside a loop, causing one query per iteration instead of a single eager-loaded query. For example:

```php
// BAD: N+1 -- runs 1 query for properties + N queries for office
@foreach($properties as $property)
    {{ $property->office->name }}  // Each iteration hits the database
@endforeach

// GOOD: Eager loaded -- runs 2 queries total
$properties = Property::with('office')->get();
@foreach($properties as $property)
    {{ $property->office->name }}  // No additional query
@endforeach
```

## Analysis Process

### Step 1: Determine Scope

- If a `[file-path]` argument is provided, analyze only that file and trace its data sources.
- If no argument is provided, scan the entire project. Use `Glob` to find:
  - `resources/views/**/*.blade.php` (Blade views)
  - `app/Http/Controllers/**/*.php` (Controllers)
  - `app/Livewire/**/*.php` (Livewire class components, if any exist)

### Step 2: Scan Blade Views for Relationship Access in Loops

For each Blade file, use `Read` to load the content and identify:

1. **Loop constructs**: `@foreach`, `@forelse`, `@for`, `@while`, and nested loops.
2. **Relationship access inside loops**: Look for patterns like:
   - `$variable->relationship` where `relationship` is a known model relationship
   - `$variable->relationship->property`
   - `$variable->relationship()->...`
   - `$variable->relationship->count()`
   - `$variable->loadCount('relationship')` inside loops (should be done before the loop)
3. **Nested relationship access**: `$property->office->organization->name` -- multiple levels of lazy loading.

To determine if a property access is a relationship:
- Read the model class associated with the loop variable.
- Check if the accessed property matches a relationship method name in that model.
- Cross-reference with the model's `$appends` array (appended attributes are not relationships even if they share a name).

### Step 3: Trace Data Sources

For each loop variable, trace where the data comes from:

1. **Volt/Livewire components**: Look at the PHP block at the top of the Blade file for:
   - Public properties assigned in `mount()` or computed properties
   - Queries in `render()` or `with()` methods
   - Check if `with()` or `load()` is called on the query
2. **Controller-passed data**: If the view is rendered by a controller:
   - Use `Grep` to find which controller renders this view (`return view('view.name', ...)`)
   - Read the controller method to find the query
   - Check for `with()`, `load()`, or `loadMissing()` calls
3. **Livewire component properties**: Check if the component eager-loads in its query.

### Step 4: Identify Missing Eager Loads

For each relationship accessed inside a loop, determine:

1. **Is the relationship eager-loaded?** Check if `with('relationship')` is called on the query that builds the collection.
2. **Is it loaded via `$with` on the model?** Some models define `protected $with = ['relationship']` for automatic eager loading.
3. **Is it loaded conditionally?** `loadMissing()` or `whenLoaded()` patterns.

Flag as N+1 only if the relationship is NOT eager-loaded by any of these mechanisms.

### Step 5: Check for Other N+1 Patterns

Beyond simple loop access, check for:

1. **Accessor N+1**: Model accessors that access relationships, called from within loops.
   ```php
   // In Model
   public function getFullAddressAttribute() {
       return $this->location->name . ', ' . $this->location->province->name; // N+1!
   }

   // In Blade
   @foreach($properties as $property)
       {{ $property->full_address }}  // Triggers N+1 via accessor
   @endforeach
   ```

2. **Relationship count N+1**: Using `$model->relationship->count()` instead of `withCount()`.
   ```php
   @foreach($properties as $property)
       {{ $property->images->count() }}  // Loads all images just to count
   @endforeach
   ```

3. **Conditional relationship access**: Relationships accessed inside `@if` blocks within loops.
   ```php
   @foreach($properties as $property)
       @if($property->showcase)  // N+1 even though it's conditional
           {{ $property->showcase->name }}
       @endif
   @endforeach
   ```

4. **Component rendering N+1**: Livewire/Blade components rendered inside loops that internally access relationships.

### Step 6: Report Findings

For each N+1 problem found, report:

```
N+1 QUERY DETECTED
===================

File: /absolute/path/to/resources/views/livewire/properties/index.blade.php
Line: 45
Loop: @foreach($properties as $property)
Access: $property->office->name

Data Source: Livewire computed property in same file (line 12)
Query: Property::where('organization_id', $this->organizationId)->paginate(20)

Problem: 'office' relationship is not eager-loaded. This will execute 1 additional
query per property in the collection (20 queries for a page of 20).

Fix: Add ->with('office') to the query:
  Property::where('organization_id', $this->organizationId)
      ->with('office')
      ->paginate(20);
```

### Step 7: Summary

End with a summary table:

```
N+1 ANALYSIS SUMMARY
======================

Files scanned:          85
Loops analyzed:        142
Relationship accesses: 234
N+1 problems found:    12

By severity:
  High (in paginated/large collections):  4
  Medium (in moderate collections):       5
  Low (in small/bounded collections):     3

Top offenders:
  resources/views/livewire/properties/index.blade.php     - 4 issues
  resources/views/livewire/clients/show.blade.php         - 3 issues
  resources/views/livewire/operations/table.blade.php     - 2 issues
```

## Fix Mode (`fix` or `fix [file-path]`)

For each N+1 problem detected, automatically add the missing `with()` calls:

1. **Locate the query** that builds the collection (in Volt PHP block, controller, or Livewire component).
2. **Add the missing relationship** to an existing `with()` call, or add a new `->with('relationship')` clause.
3. **For relationship count issues**, replace `->count()` access with `withCount()` on the query.
4. **Ask for confirmation** before each change.
5. Show the before/after diff for the query modification.

## Dry Run Mode (`fix --dry-run`)

Show exactly what query modifications would be made, with before/after diffs, without writing any changes.

## Severity Classification

- **High**: Relationship accessed in a loop over a paginated or potentially large collection (could be hundreds of extra queries).
- **Medium**: Relationship accessed in a loop over a moderate collection (10-50 items typically).
- **Low**: Relationship accessed in a loop over a small, bounded collection (e.g., `->take(5)`, enum values, etc.).

## Notes

- This is **static analysis only**. It cannot detect dynamically constructed queries or runtime-conditional eager loading.
- False positives may occur when a property name matches a relationship name but is actually an attribute or appended value.
- Polymorphic relationships (`morphTo`) are harder to trace -- note these as "needs manual review" if uncertain.
- The `$with` model property provides automatic eager loading and should be checked before flagging.
