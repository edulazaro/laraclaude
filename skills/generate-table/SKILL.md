---
name: lc:generate-table
description: Generate a table component with filters, sorting, infinite scroll using the project's TableComponent base.
argument-hint: "[ModelName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `<ModelName>` | Generate a table Volt component for the given model (e.g., `Property`, `Client`, `User`, `Invoice`) |

## Overview

This is a generator skill. It creates a complete table Volt component with filtering, sorting, infinite scroll, and row actions, following the project's established TableComponent pattern and UI conventions.

## Instructions

### Step 1: Understand the project's table patterns

Before generating, read existing table implementations to match conventions exactly:

1. Read the base `TableComponent` class:
   - Search for `class TableComponent` using Grep
   - Understand: base properties, methods (`loadMore`, `loadInitial`, `sortBy`), traits used

2. Read 1-2 existing table components for reference:
   - Use Glob: `resources/views/livewire/**/*.blade.php`
   - Look for components that extend `TableComponent`
   - Note the exact structure, column definitions, filter setup, and action dropdowns

3. Read the table-related Blade components:
   - `resources/views/components/tables/table.blade.php` (or search for `x-tables.table`)
   - `resources/views/components/tables/dropdown.blade.php` (row action dropdown)
   - `resources/views/components/filters.blade.php` (or search for `x-filters`)

### Step 2: Parse the argument and determine paths

Convert the ModelName to file paths:

- `Property` -> `resources/views/livewire/properties/index.blade.php`
- `Client` -> `resources/views/livewire/clients/index.blade.php`
- `Invoice` -> `resources/views/livewire/invoices/index.blade.php`

Pluralize the model name for the directory. Use `index.blade.php` as the filename.

### Step 3: Inspect the model

Read the model file to understand:
- Table name and fillable fields
- Available relationships (for eager loading)
- Scopes that could be useful for filtering
- Casts (dates, enums, booleans) that affect display
- Soft deletes (if present, may need trash filter)

```
app/Models/ModelName.php
```

### Step 4: Ask the user (if needed)

If the model has many fields, ask which columns to display in the table. If the context makes it clear, infer sensible defaults:
- Show 4-7 columns maximum
- Prioritize: name/title, status, key relationships, dates
- Always include a date column (created_at or relevant date)

### Step 5: Generate the Volt component

Create the file using Write tool.

**Template structure**:

```php
<?php

use App\Models\ModelName;
use App\Http\Controllers\TableComponent;
use Livewire\Volt\Component;

new class extends TableComponent {

    public string $search = '';
    public string $sortField = 'created_at';
    public string $sortDirection = 'desc';
    public string $statusFilter = '';

    protected function query()
    {
        $query = ModelName::query()
            ->with(['relationship1', 'relationship2']);

        if ($this->search) {
            $query->where(function ($q) {
                $q->where('name', 'like', '%' . $this->search . '%')
                  ->orWhere('email', 'like', '%' . $this->search . '%');
            });
        }

        if ($this->statusFilter) {
            $query->where('status', $this->statusFilter);
        }

        return $query->orderBy($this->sortField, $this->sortDirection);
    }
}; ?>

<div>
    <x-filters>
        <x-filters.search wire:model.live.debounce.300ms="search" />

        <x-filters.select wire:model.live="statusFilter">
            <option value="">@text('all_statuses', 'All Statuses')</option>
            <option value="active">@text('active', 'Active')</option>
            <option value="inactive">@text('inactive', 'Inactive')</option>
        </x-filters.select>
    </x-filters>

    <x-tables.table
        :columns="[
            text('name', 'Name'),
            text('email', 'Email'),
            text('status', 'Status'),
            text('created_at', 'Created'),
        ]"
        :sortField="$sortField"
        :sortDirection="$sortDirection"
        :sortableColumns="['name', 'email', 'status', 'created_at']"
    >
        @forelse($this->items as $item)
            <tr wire:key="model-{{ $item->id }}">
                <td class="px-4 py-3 text-sm text-gray-900 dark:text-white">
                    {{ $item->name }}
                </td>
                <td class="px-4 py-3 text-sm text-gray-500 dark:text-gray-400">
                    {{ $item->email }}
                </td>
                <td class="px-4 py-3 text-sm">
                    <span class="px-2 py-1 text-xs font-medium rounded-full
                        {{ $item->status === 'active'
                            ? 'bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-300'
                            : 'bg-gray-100 text-gray-800 dark:bg-gray-700 dark:text-gray-300' }}">
                        {{ $item->status }}
                    </span>
                </td>
                <td class="px-4 py-3 text-sm text-gray-500 dark:text-gray-400">
                    {{ $item->created_at->format('d/m/Y') }}
                </td>
                <td class="px-4 py-3 text-sm text-right">
                    <x-tables.dropdown>
                        <x-tables.dropdown-item
                            href="{{ route('models.show', $item) }}"
                            wire:navigate
                        >
                            @text('view', 'View')
                        </x-tables.dropdown-item>
                        <x-tables.dropdown-item
                            @click="$dispatch('open-edit-model', { id: {{ $item->id }} })"
                        >
                            @text('edit', 'Edit')
                        </x-tables.dropdown-item>
                        <x-tables.dropdown-item
                            @click="$dispatch('confirm-delete-model', { id: {{ $item->id }} })"
                            class="text-red-600 dark:text-red-400"
                        >
                            @text('delete', 'Delete')
                        </x-tables.dropdown-item>
                    </x-tables.dropdown>
                </td>
            </tr>
        @empty
            <tr>
                <td colspan="5" class="px-4 py-8 text-center text-gray-500 dark:text-gray-400">
                    @text('no_records_found', 'No records found')
                </td>
            </tr>
        @endforelse
    </x-tables.table>

    <div x-intersect="$wire.loadMore()">
    </div>
</div>
```

### Step 6: Key conventions to follow

1. **NEVER include "Actions" in the columns array** - The actions column is automatically handled by `x-tables.dropdown`. Only list data columns in the `:columns` array.

2. **ALWAYS use `wire:key` on each `<tr>`** - Use a unique key like `model-{{ $item->id }}` to prevent DOM diffing bugs.

3. **Use `@text()` for all user-visible text** - Column headers, status labels, filter labels, empty state messages. Defaults must be English.

4. **Infinite scroll with `x-intersect`** - Place `<div x-intersect="$wire.loadMore()">` after the table for automatic loading.

5. **Filters use `wire:model.live.debounce.300ms`** for search and `wire:model.live` for selects.

6. **Eager load relationships** - Always use `->with([...])` in the query to avoid N+1 problems.

7. **Dark mode support** - Every visual element must have both light and dark variants.

8. **Volt single-file pattern** - PHP at top, blade below, never separate class files.

9. **Model imports** - Use `use App\Models\ModelName;` at the top.

10. **Sort defaults** - Default sort by `created_at desc` unless another field makes more sense for the model.

### Step 7: Adapt to the existing TableComponent

The generated component MUST match the exact interface of the project's `TableComponent`. After reading the base class in Step 1:

- Match the exact method signatures (`query()`, `loadMore()`, `loadInitial()`, etc.)
- Use the correct property names for items (might be `$items`, `$this->items`, or a computed property)
- Follow the same pagination/infinite scroll mechanism
- Match any trait usage

If the base class uses a different pattern than shown in the template above, adapt the template to match.

### Step 8: Show usage instructions

After generating, display:

```
## Generated: resources/views/livewire/invoices/index.blade.php

### Add route in routes/app.php:
Route::get('/invoices', function () {
    return view('invoices.index');
})->name('invoices.index');

### Create the wrapper view (resources/views/invoices/index.blade.php):
<x-app-layout>
    <livewire:invoices.index />
</x-app-layout>

### Or use directly in another component:
<livewire:invoices.index />

### Columns displayed: name, email, status, created_at
### Filters: search (name, email), status
### Sorting: all columns sortable
### Actions: view, edit (opens modal), delete (opens confirmation)
```
