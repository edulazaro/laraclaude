---
name: lc:livewire-optimize
description: Detect and fix Livewire performance issues - heavy renders, excessive queries, unnecessary reactivity.
argument-hint: "[report | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `report` | Scan all Volt components and report performance issues without changes |
| `fix` | Scan all Volt components, report issues, and auto-fix what is safe |
| `fix --dry-run` | Show what fixes would be applied without writing changes |
| `<file-path>` | Scan a single file and report performance issues |
| `fix <file-path>` | Scan a single file and auto-fix performance issues |

## Overview

This skill analyzes Livewire/Volt components for performance anti-patterns that cause slow page loads, excessive server requests, large payloads, and unnecessary re-renders. It focuses on issues that degrade user experience at scale.

## Instructions

### Step 1: Determine scope

Parse the argument:

- **No argument or `report`**: Scan all Volt components under `resources/views/livewire/`.
- **`fix` (no path)**: Same scope, apply safe fixes.
- **`fix --dry-run`**: Show proposed fixes without applying.
- **`<file-path>`**: Single file report.
- **`fix <file-path>`**: Single file fix.

Use Glob to find Volt components:
```
resources/views/livewire/**/*.blade.php
```

Filter to files containing `new class extends Component`.

### Step 2: Run each detection rule

#### Rule 1: Queries inside render() or blade template

**Pattern to detect**: Database queries executed during render. Look for:

- `Model::where(`, `Model::find(`, `DB::table(`, `DB::select(` inside the `render()` method
- `->get()`, `->first()`, `->count()`, `->pluck()`, `->paginate()` inside `render()`
- Queries inside blade template via `{{ Model::where(...) }}` or `@php` blocks
- Relationship access that triggers lazy loading in blade: `$this->model->relation->` where `relation` was not eager-loaded

These queries run on EVERY re-render (every Livewire request).

**Report format**:
```
[RENDER-QUERY] file.blade.php:89
  Query `User::where('active', true)->get()` inside render() runs on every re-render.
  Move to mount() and store result, or use #[Computed] with cache.
```

**Fix**: Move query to a `#[Computed]` method with appropriate caching, or move to `mount()` if the data is static.

#### Rule 2: Excessive public properties (payload bloat)

**Pattern to detect**: Volt components with more than 15 public properties.

Count all `public $prop` declarations in the PHP class block. Each public property is serialized and sent in every Livewire request/response.

**Report format**:
```
[EXCESSIVE-PROPS] file.blade.php
  Component has 23 public properties. Each is serialized on every request.
  Consider grouping related properties into a public array, using form objects,
  or splitting into sub-components.
```

**Fix**: No auto-fix (requires architectural decision). Report with specific suggestions based on the property types found.

#### Rule 3: wire:poll with heavy components

**Pattern to detect**: `wire:poll` or `wire:poll.Xs` on components that:
- Have more than 10 public properties
- Contain queries in render()
- Have a polling interval under 5 seconds

**Report format**:
```
[HEAVY-POLL] file.blade.php:12
  wire:poll.2s on component with 18 public properties and render() queries.
  Each poll sends full component state. Consider:
  - Increasing interval to 10s+
  - Using wire:poll.visible to pause when tab is hidden
  - Extracting polled data to a smaller sub-component
```

**Fix**: Add `.visible` modifier if missing. Suggest increasing interval in report.

#### Rule 4: Large arrays/collections as public properties

**Pattern to detect**: Public properties that are assigned large datasets:
- `$this->items = Model::all()->toArray()` or `Model::get()->toArray()` (unbounded queries)
- Array properties populated in mount() from queries without `->limit()` or `->take()`
- Properties assigned from `->pluck()->toArray()` on tables known to have many rows

**Report format**:
```
[LARGE-PAYLOAD] file.blade.php:34
  Public property `$allLocations` assigned from `Location::all()->toArray()`.
  This could be thousands of items serialized on every request.
  Consider: server-side search, pagination, or lazy loading.
```

**Fix**: No auto-fix. Report only.

#### Rule 5: Components missing lazy loading

**Pattern to detect**: `<livewire:component-name>` tags in blade templates (parent components) where the child component:
- Is below the fold (inside tabs, accordions, or far down in the layout)
- Has heavy mount() logic (queries, file operations)
- Is inside `@if` blocks that may not be initially visible

Look for child components rendered inside:
- Tab panels (`role="tabpanel"`, `x-show`, conditional display)
- Accordion bodies
- Modal bodies
- Elements with `hidden` or `display:none` initial state

**Report format**:
```
[MISSING-LAZY] parent.blade.php:156
  <livewire:properties.features> is inside a hidden tab panel.
  Use <livewire:properties.features lazy /> to defer loading until visible.
```

**Fix**: Add `lazy` attribute to the `<livewire:` tag.

#### Rule 6: Missing #[Computed] on repeated method calls

**Pattern to detect**: Methods in the PHP class block that are called multiple times in the blade template without `#[Computed]` attribute.

Look for:
- `$this->methodName()` appearing 2+ times in the blade section
- The method in the PHP block does NOT have `#[Computed]` attribute
- The method performs work (queries, calculations, array operations) - not simple getters

**Report format**:
```
[MISSING-COMPUTED] file.blade.php
  Method `getVisibleFields()` called 4 times in template without #[Computed].
  Each call executes the method body. Add #[Computed] to cache within request.
```

**Fix**: Add `#[Computed]` attribute above the method. Add `use Livewire\Attributes\Computed;` import if not present. Rename method to remove `get` prefix if following Livewire 3 conventions (e.g., `getVisibleFields()` -> `visibleFields()` with `$this->visibleFields` access in blade).

#### Rule 7: Multiple wire:model.live in same form

**Pattern to detect**: `<form>` elements (or containers with `wire:submit`) containing more than 2 `wire:model.live` bindings without debounce.

Each `wire:model.live` field sends a server request on every input change. Multiple such fields in one form compound the problem.

**Report format**:
```
[EXCESSIVE-LIVE-BINDING] file.blade.php:45-89
  Form contains 5 wire:model.live bindings without debounce.
  Each keystroke on any field triggers a full server roundtrip.
  Consider:
  - Using wire:model (update on submit only) for most fields
  - Adding .debounce.300ms for fields that need live updates
  - Using wire:model.blur for fields that only need update on focus loss
```

**Fix**: Convert `wire:model.live` to `wire:model` for fields that don't need real-time updates. Keep `wire:model.live.debounce.300ms` for search/filter fields.

#### Rule 8: Monolithic components that should be split

**Pattern to detect**: Components where:
- The PHP class block exceeds 200 lines
- There are more than 10 public methods (excluding lifecycle hooks)
- The blade template exceeds 400 lines
- Multiple unrelated concerns are handled (e.g., CRUD + filters + modals + exports)

**Report format**:
```
[MONOLITHIC-COMPONENT] file.blade.php
  Component has 285 PHP lines, 18 methods, 520 template lines.
  Consider splitting into:
  - A parent coordinator component
  - Separate filter component
  - Separate table/list component
  - Separate modal components
```

**Fix**: No auto-fix. Report with specific splitting suggestions based on the component's structure.

### Step 3: Generate report

Group findings by severity and impact:

**Severity levels**:
- **CRITICAL**: Causes visible performance degradation (render queries, heavy polling, large payloads)
- **WARNING**: Wastes resources but may not be immediately visible (excessive props, missing lazy, missing computed)
- **INFO**: Best practices that improve performance marginally (live bindings, monolithic components)

**Output format**:
```
## Livewire Performance Report

### resources/views/livewire/properties/index.blade.php

- [CRITICAL] Line 89: Query inside render() - User::where('active', true)->get()
- [CRITICAL] Line 12: wire:poll.2s with 18 public properties
- [WARNING] Component has 23 public properties (payload bloat)
- [INFO] 5 wire:model.live bindings in form without debounce

### resources/views/livewire/calendar.blade.php

- [WARNING] Method getVisibleFields() called 4x without #[Computed]
- [WARNING] <livewire:event-list> in hidden tab without lazy

---
**Summary**: 2 critical, 3 warnings, 1 info across 2 files.
**Estimated impact**: Fixing critical issues could reduce server load by ~60% on affected pages.
```

### Step 4: Apply fixes (if fix mode)

**Safe auto-fixes** (applied automatically):
- Add `#[Computed]` to methods called multiple times in blade
- Add `use Livewire\Attributes\Computed;` import
- Add `lazy` to child components in hidden containers
- Add `.visible` to `wire:poll` directives
- Convert `wire:model.live` to `wire:model` in forms (except search/filter fields)
- Add `.debounce.300ms` to remaining `wire:model.live` bindings

**Unsafe fixes** (report only):
- Moving queries out of render() (changes component behavior)
- Reducing public properties (requires architectural changes)
- Splitting monolithic components (major refactor)
- Changing poll intervals (may affect functionality)
- Removing large array properties (requires alternative data loading strategy)

For each safe fix, show before/after context. For `--dry-run`, display but do not apply.
