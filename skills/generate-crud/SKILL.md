---
name: lc:generate-crud
description: Generate complete CRUD scaffolding - migration, model, controller, routes, views (index + create/edit modal).
argument-hint: "[ModelName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `<ModelName>` | Generate full CRUD scaffolding for the given model (e.g., `Invoice`, `Task`, `Document`) |

## Overview

This is a generator skill. It creates a complete CRUD setup following the project's conventions: migration, model, controller, routes, Volt table component (index), create/edit modal, and delete confirmation modal. Every generated file follows the project's established patterns.

## Instructions

### Step 1: Gather information

Ask the user for the following (or infer from context):

1. **Model name** (required - from argument)
2. **Fields** - What columns does the table need? (name, type, nullable, default)
3. **Relationships** - Does it belong to User, Organization, Office, etc.?
4. **Multi-tenant** - Does it need `organization_id` and `office_id`? (default: yes for most models)
5. **Soft deletes** - Should records be soft-deleted? (default: yes)
6. **Which fields are displayed in the table** - Usually 4-7 columns
7. **Which fields are editable in the modal** - Usually all non-system fields

If the user provides enough context, infer sensible defaults and proceed.

### Step 2: Study existing patterns

Before generating ANY files, read existing implementations to match conventions exactly:

1. **Read an existing model** for patterns:
   ```
   app/Models/Property.php or app/Models/Client.php
   ```
   Note: fillable, casts, relationships, traits, soft deletes, slug usage

2. **Read an existing migration** for conventions:
   ```
   database/migrations/ (find a recent one)
   ```
   Note: foreign key patterns, index names, column ordering

3. **Read the base TableComponent**:
   Search for `class TableComponent` to understand the exact interface

4. **Read an existing table Volt component**:
   ```
   resources/views/livewire/ (find one extending TableComponent)
   ```

5. **Read an existing modal Volt component**:
   ```
   resources/views/livewire/modals/ (any existing modal)
   ```

6. **Read routes/app.php** for route registration patterns

7. **Read an existing controller** for the standard pattern

### Step 3: Generate the migration

Create migration file using artisan via Docker:

```bash
docker exec abodara_app php artisan make:migration create_model_names_table
```

Then edit the generated file to add the schema.

**Migration conventions**:

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('model_names', function (Blueprint $table) {
            $table->id();
            $table->string('slug')->unique();
            $table->foreignId('organization_id')->constrained()->onDelete('cascade');
            $table->foreignId('office_id')->nullable()->constrained()->onDelete('set null');
            $table->foreignId('user_id')->nullable()->constrained()->onDelete('set null');
            // ... model-specific fields
            $table->string('name');
            $table->text('description')->nullable();
            $table->enum('status', ['draft', 'active', 'archived'])->default('draft');
            $table->boolean('active')->default(true);
            $table->timestamps();
            $table->softDeletes();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('model_names');
    }
};
```

**Key migration rules**:
- Always include `slug` with unique index for URL-friendly identifiers
- Use `foreignId()->constrained()` for relationships
- Organization/office foreign keys: organization cascades, office sets null
- Put `timestamps()` and `softDeletes()` last
- Use `enum` for finite status sets
- Use `decimal(12, 2)` for monetary values
- Use `text` for long content, `string` for short content

### Step 4: Generate the model

Create file at `app/Models/ModelName.php`.

**Model conventions**:

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\HasMany;

class ModelName extends Model
{
    use SoftDeletes;

    protected $fillable = [
        'slug',
        'organization_id',
        'office_id',
        'user_id',
        'name',
        'description',
        'status',
        'active',
    ];

    protected $casts = [
        'active' => 'boolean',
    ];

    public function organization(): BelongsTo
    {
        return $this->belongsTo(Organization::class);
    }

    public function office(): BelongsTo
    {
        return $this->belongsTo(Office::class);
    }

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }
}
```

**Key model rules**:
- Always import related models with `use` statements
- Use return type hints on relationships (`: BelongsTo`, `: HasMany`)
- Include `SoftDeletes` trait when migration has `softDeletes()`
- Cast booleans, dates, and enums appropriately
- No comments unless explicitly requested
- Fillable array includes all user-editable fields
- Do NOT include `id`, `created_at`, `updated_at`, `deleted_at` in fillable

### Step 5: Generate the controller

Create file at `app/Http/Controllers/ModelNameController.php`.

**Controller conventions**:

```php
<?php

namespace App\Http\Controllers;

use App\Models\ModelName;
use Illuminate\Http\Request;

class ModelNameController extends Controller
{
    public function index()
    {
        return view('model-names.index');
    }

    public function show(ModelName $modelName)
    {
        return view('model-names.show', compact('modelName'));
    }
}
```

**Key controller rules**:
- Keep controllers thin - logic belongs in Volt components, actions, or services
- Use route model binding
- Return views that contain Livewire components
- No CRUD methods in controller (handled by Livewire modals)

### Step 6: Generate the wrapper view

Create `resources/views/model-names/index.blade.php`:

```blade
<x-app-layout>
    <livewire:model-names.index />
</x-app-layout>
```

### Step 7: Generate the table Volt component

Create `resources/views/livewire/model-names/index.blade.php`.

Follow the EXACT same pattern as the `lc:generate-table` skill. Key points:

- Extend `TableComponent`
- Include search filter with debounce
- Include status filter if model has status field
- Sortable columns
- `wire:key` on every row
- `x-tables.dropdown` for row actions (NOT in columns array)
- Infinite scroll with `x-intersect`
- `@text()` for all visible strings
- Dark mode support
- Empty state

Include the modals at the bottom of the component:

```blade
<div>
    {{-- Filters and table content --}}

    {{-- Modals --}}
    <livewire:modals.create-model-name />
    <livewire:modals.edit-model-name />
    <livewire:modals.confirm-delete-model-name />
</div>
```

### Step 8: Generate the create/edit modal

Create `resources/views/livewire/modals/create-model-name.blade.php` and `resources/views/livewire/modals/edit-model-name.blade.php`.

Follow the EXACT same pattern as the `lc:generate-modal` skill. Key points:

- Volt single-file component
- `<x-modal>` with proper id and title
- `x-fields.*` components for inputs (already include @error - NO duplicate error display)
- Footer slot with cancel and submit buttons
- Validation rules
- Event dispatching for parent refresh
- Reset form after successful action

For the **create** modal:
- Generate a slug using `Str::uuid()` or `Str::slug()`
- Set `organization_id` and `office_id` from the authenticated user's context
- Dispatch `model-name-created` event after creation

For the **edit** modal:
- Load data via `#[On('open-edit-model-name')]` event
- Populate form fields from model
- Dispatch `model-name-updated` event after update

### Step 9: Generate the delete confirmation modal

Create `resources/views/livewire/modals/confirm-delete-model-name.blade.php`.

Use the confirmation modal variant from `lc:generate-modal`. Dispatch `model-name-deleted` event.

### Step 10: Register routes

Edit `routes/app.php` to add the new routes.

Read the file first to find the right location, then add:

```php
Route::get('/model-names', [ModelNameController::class, 'index'])->name('model-names.index');
Route::get('/model-names/{modelName}', [ModelNameController::class, 'show'])->name('model-names.show');
```

Add the `use` import for the controller at the top of the routes file.

### Step 11: Run migration

```bash
docker exec abodara_app php artisan migrate
```

### Step 12: Summary output

After generating all files, display a complete summary:

```
## CRUD Generated: ModelName

### Files created:
1. database/migrations/YYYY_MM_DD_HHMMSS_create_model_names_table.php
2. app/Models/ModelName.php
3. app/Http/Controllers/ModelNameController.php
4. resources/views/model-names/index.blade.php
5. resources/views/livewire/model-names/index.blade.php
6. resources/views/livewire/modals/create-model-name.blade.php
7. resources/views/livewire/modals/edit-model-name.blade.php
8. resources/views/livewire/modals/confirm-delete-model-name.blade.php

### Routes added to routes/app.php:
- GET /model-names -> model-names.index
- GET /model-names/{modelName} -> model-names.show

### Migration status:
Migration executed successfully.

### Next steps:
- Add navigation link to sidebar/menu
- Add authorization policies if needed
- Customize table columns and filters
- Add any additional business logic to actions
```

### Key conventions across ALL generated files

1. **Volt single-file components** - NEVER create separate Livewire PHP class files
2. **`@text()` for all user-visible text** - Defaults in English, no hardcoded strings
3. **Model imports with `use`** - Never inline `\App\Models\`
4. **No comments** - Unless explicitly requested
5. **Pink color scheme** for primary buttons (`bg-pink-600 hover:bg-pink-700`)
6. **Dark mode** on every visual element
7. **`wire:navigate`** on internal links
8. **Soft deletes** by default
9. **Slug field** for URL-friendly identifiers
10. **Multi-tenant** with `organization_id` by default
