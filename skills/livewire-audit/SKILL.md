---
name: lc:livewire-audit
description: Detect common Livewire/Volt problems - unserializable properties, missing wire:key, orphaned events, re-render issues.
argument-hint: "[report | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `report` | Scan all Volt components and report issues without making changes |
| `fix` | Scan all Volt components, report issues, and auto-fix what is safe to fix |
| `fix --dry-run` | Show what fixes would be applied without writing changes |
| `<file-path>` | Scan a single file and report issues |
| `fix <file-path>` | Scan a single file and auto-fix issues |

## Overview

This skill audits Livewire/Volt components for common bugs, anti-patterns, and runtime errors. It targets issues that cause serialization failures, DOM diffing bugs, broken event wiring, and silent rendering problems.

## Instructions

### Step 1: Determine scope

Parse the argument to decide what to scan:

- **No argument or `report`**: Scan all `.blade.php` files under `resources/views/livewire/` that contain Volt component signatures (`new class extends Component`).
- **`fix` (no path)**: Same scope as `report` but apply fixes.
- **`fix --dry-run`**: Same scope as `fix` but only show proposed changes.
- **`<file-path>`**: Scan only the specified file.
- **`fix <file-path>`**: Scan and fix only the specified file.

Use Glob to find files:
```
resources/views/livewire/**/*.blade.php
```

Filter to only files containing Volt component PHP blocks (look for `new class extends Component`).

### Step 2: Run each detection rule

For every file in scope, check the following rules in order:

#### Rule 1: Unserializable public properties

Livewire can only serialize: primitives (string, int, float, bool, null, array), Eloquent Models, Eloquent Collections, Carbon instances, Stringable, and Enums.

**Pattern to detect**: Public properties with type hints or assignments that are objects NOT in the safe list.

Look for:
- `public SomeClass $prop` where `SomeClass` is not a Model, Collection, Carbon, BackedEnum, or Stringable
- `$this->prop = new SomeObject()` in mount/methods where `$prop` is public and `SomeObject` is not safe
- Public properties assigned closures or anonymous functions

**Report format**:
```
[UNSERIALIZABLE] file.blade.php:42
  Public property `$fieldConfig` has type `FieldConfig` which cannot be serialized by Livewire.
  Suggestion: Make it protected and use a getter method, or convert to array.
```

**Fix**: No auto-fix (requires manual refactoring). Report only.

#### Rule 2: Missing wire:key in @foreach loops rendering Livewire components

**Pattern to detect**: `@foreach` loops that contain `<livewire:` or `@livewire(` tags without a `wire:key` or `:wire:key` attribute.

Look for blocks like:
```blade
@foreach($items as $item)
    <livewire:some-component :model="$item" />
@endforeach
```

Where the `<livewire:` tag does NOT have `wire:key="..."` or `:wire:key="..."`.

Also check for `@foreach` loops containing `<div>` or other elements wrapping Livewire components without `wire:key` on either the wrapper or the component.

**Report format**:
```
[MISSING-WIRE-KEY] file.blade.php:87
  @foreach loop renders <livewire:card> without wire:key.
  This causes DOM diffing bugs when items are reordered or removed.
```

**Fix**: Add `wire:key="{{ $loop->index }}"` to the Livewire component tag. Prefer using a unique model identifier if available (e.g., `wire:key="component-{{ $item->id }}"`).

#### Rule 3: wire:model referencing non-existent public properties

**Pattern to detect**: `wire:model="fieldName"` or `wire:model.live="fieldName"` in the blade section where `fieldName` (or the root property if dot notation) is not declared as a `public` property in the PHP block.

Parse the PHP class block to extract all public property names. Then scan the blade template for `wire:model` bindings and check each root property name exists.

Handle dot notation: `wire:model="data.name"` means the root property is `$data`.

**Report format**:
```
[MISSING-PROPERTY] file.blade.php:123
  wire:model="clientName" but no public property `$clientName` found in component.
```

**Fix**: No auto-fix (the developer must decide the property type and default). Report only.

#### Rule 4: Orphaned dispatched events

**Pattern to detect**: `$this->dispatch('event-name')` or `$dispatch('event-name')` where no component in the project has a matching `#[On('event-name')]` listener.

Exclude well-known framework events: `close-modal`, `open-modal`, `close-drawer`, `open-drawer`, `notify`, `toast`, `refresh`.

Search across ALL Volt components (not just the current file) for `#[On('event-name')]` attributes.

**Report format**:
```
[ORPHANED-EVENT] file.blade.php:56
  Dispatches 'property-updated' but no component has #[On('property-updated')] listener.
```

**Fix**: No auto-fix. Report only.

#### Rule 5: Missing wire:ignore for JS-manipulated DOM inside Livewire components

**Pattern to detect**: Elements with `id` attributes that are referenced by JavaScript (`document.getElementById`, `document.querySelector('#id')`) inside or alongside the component, where the parent does not have `wire:ignore`.

Also detect: Elements inside `<x-modal>` or drawer components that are populated by JavaScript event listeners (look for `addEventListener` patterns that set `textContent`, `innerHTML`, or `value`).

**Report format**:
```
[MISSING-WIRE-IGNORE] file.blade.php:145
  Element #eventTitle is updated by JavaScript but not inside a wire:ignore block.
  Livewire will reset this content on re-render.
```

**Fix**: Wrap the affected elements in a `<div wire:ignore>` container.

#### Rule 6: wire:model.live without debounce on potentially expensive operations

**Pattern to detect**: `wire:model.live="prop"` where the component has methods triggered by that property update (e.g., `updated` hooks, computed properties that query the database, or the property is used in a query inside `render()`).

Also flag: multiple `wire:model.live` bindings in the same form (each keystroke on any field = full server roundtrip).

**Report format**:
```
[MISSING-DEBOUNCE] file.blade.php:78
  wire:model.live="search" triggers server request on every keystroke.
  Consider wire:model.live.debounce.300ms="search".
```

**Fix**: Replace `wire:model.live="prop"` with `wire:model.live.debounce.300ms="prop"`.

#### Rule 7: Missing wire:navigate on internal links

**Pattern to detect**: `<a href="...">` tags pointing to internal routes (containing `route('...')` or relative paths starting with `/`) that do NOT have `wire:navigate` attribute.

Exclude: links with `target="_blank"`, links to external URLs, anchor links (`#`), links to file downloads, links already having `wire:navigate`.

Only flag this in files that use an SPA layout (check if the layout extends a layout with `@livewireScripts` or uses `wire:navigate` elsewhere).

**Report format**:
```
[MISSING-WIRE-NAVIGATE] file.blade.php:34
  Internal link <a href="{{ route('properties.index') }}"> missing wire:navigate.
```

**Fix**: Add `wire:navigate` attribute to the `<a>` tag.

#### Rule 8: Public Eloquent Collections

**Pattern to detect**: Public properties typed as `\Illuminate\Database\Eloquent\Collection` or `\Illuminate\Support\Collection`, or assigned from `->get()`, `->all()`, or `Collection::make()`.

Large collections as public properties increase payload size on every Livewire request.

**Report format**:
```
[COLLECTION-PROPERTY] file.blade.php:15
  Public property `$users` is an Eloquent Collection. Consider using ->toArray(),
  a computed property, or pagination to reduce payload size.
```

**Fix**: No auto-fix. Report only with suggestion.

#### Rule 9: Redirect with navigate:true to external URLs

**Pattern to detect**: `$this->redirect('http...', navigate: true)` or `$this->redirectRoute('...', navigate: true)` where the URL is an external domain.

**Report format**:
```
[BAD-REDIRECT] file.blade.php:67
  $this->redirect() with navigate:true to external URL will fail.
  Remove navigate:true for external redirects.
```

**Fix**: Remove `navigate: true` from the redirect call.

### Step 3: Generate report

Group findings by file, then by severity:

**Severity levels**:
- **ERROR**: Will cause runtime failures (unserializable, bad redirect, missing property)
- **WARNING**: Will cause bugs or unexpected behavior (missing wire:key, orphaned events, missing wire:ignore)
- **INFO**: Performance or best-practice suggestions (debounce, wire:navigate, collections)

**Output format**:
```
## Livewire Audit Report

### resources/views/livewire/calendar.blade.php

- [ERROR] Line 42: Unserializable public property `$fieldConfig` (FieldConfig)
- [WARNING] Line 87: @foreach renders <livewire:card> without wire:key
- [INFO] Line 78: wire:model.live="search" should use debounce

### resources/views/livewire/properties/index.blade.php

- [WARNING] Line 34: Internal link missing wire:navigate

---
**Summary**: 2 errors, 2 warnings, 1 info across 2 files.
```

### Step 4: Apply fixes (if fix mode)

For `fix` or `fix <file-path>`:
1. Show each proposed change with before/after context
2. Apply the change using Edit tool
3. Report what was changed

For `fix --dry-run`:
1. Show each proposed change with before/after context
2. Do NOT apply changes
3. Report what would be changed

**Safe auto-fixes** (applied automatically):
- Add `wire:key` to Livewire components in loops
- Add `debounce.300ms` to `wire:model.live`
- Add `wire:navigate` to internal links
- Remove `navigate: true` from external redirects
- Wrap JS-manipulated elements in `wire:ignore`

**Unsafe fixes** (report only, never auto-fix):
- Unserializable properties (requires refactoring)
- Missing public properties for wire:model (developer decision)
- Orphaned events (may be cross-component or in JS)
- Collection properties (multiple valid solutions)
