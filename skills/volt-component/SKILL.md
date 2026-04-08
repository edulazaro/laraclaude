---
name: lc:volt-component
description: Generate a Livewire Volt single-file component following project conventions.
argument-hint: "[component-name]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Generate Volt Component

Create a Livewire Volt single-file component that follows the project's conventions and patterns. All Livewire components in this project MUST use the Volt single-file format -- never create separate PHP class files in `app/Livewire/`.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for the component name and purpose, then generate. |
| `[component-name]` | Generate a Volt component at the specified path. Names can use dot or slash notation. |

This is a generator skill -- it always creates files. There is no analyze-only or fix mode.

## Process

### Step 1: Determine Component Details

1. If a `[component-name]` argument is provided, use it as the component name.
   - Names can use dot notation (e.g., `properties.edit`) or slash notation (e.g., `properties/edit`).
   - The file will be created at `resources/views/livewire/{name}.blade.php` (dots become directory separators).
2. If the component purpose is not clear from the name, ask the user what the component should do before generating code.
3. Check if a file already exists at the target path. If it does, warn the user and ask if they want to overwrite.

### Step 2: Study Existing Patterns

Before generating, examine existing components to match the project's conventions:

1. Use `Glob` to find similar components in the same directory or feature area.
2. Use `Read` to examine 1-2 existing components to understand:
   - Import patterns (which models, facades, traits are commonly used)
   - How mount() is structured
   - Validation patterns
   - Flash message patterns
   - How the template is structured (Tailwind classes, component usage)
3. Check the CLAUDE.md conventions (already loaded in context):
   - Use `@text()` for all translations with English defaults
   - Single root element
   - Support dark mode
   - Use pink color scheme for primary actions (or dynamic `$color` in group context)
   - Use `wire:model` (not `.live` unless real-time updates needed)
   - Use `<x-modal>` for modals
   - No comments unless requested

### Step 3: Generate the Component

Create the file using `Write` with this structure:

```php
<?php

use App\Models\RequiredModel;
use Illuminate\Support\Facades\Auth;
use Livewire\Volt\Component;

new class extends Component {
    // Public properties for data binding
    public $property;

    // Form data properties
    public string $name = '';
    public string $email = '';

    // Validation rules
    protected function rules(): array
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email',
        ];
    }

    public function mount($property = null)
    {
        // Initialize component state
        if ($property) {
            $this->property = $property;
            $this->name = $property->name;
        }
    }

    public function save()
    {
        $this->validate();

        // Business logic here

        session()->flash('success', text('saved_successfully', 'Saved successfully'));
    }
}; ?>

<div>
    {{-- Template content --}}
</div>
```

### Component Patterns

#### Basic CRUD Component
For components that manage a single model (create/edit):
- Properties matching the model's fillable fields
- `mount()` to load existing data for edit mode
- `save()` method with validation
- Flash messages for success/error feedback

#### Table/List Component
For components that display collections:
- Search/filter properties with `wire:model.live`
- Pagination support
- Computed property or method for the query with eager loading
- Sort functionality

#### Modal Trigger Component
For components that open modals:
- Use `$dispatch('open-modal', { id: 'modal-id' })` for opening
- Use `$dispatch('close-modal')` for closing
- Listener methods for modal events

#### Form Component
For standalone form components:
- Validation rules as method (not property, for dynamic rules)
- `$this->reset()` after successful submission
- Proper error display via `@error` directives on `x-fields` components

### Step 4: Conventions Checklist

Before writing the file, verify:

- [ ] PHP block at top with `new class extends Component`
- [ ] All model imports use `use` statements (never inline `\App\Models\...`)
- [ ] Single root `<div>` element wrapping all template content
- [ ] All user-visible text wrapped in `@text('key', 'Default')` or `text('key', 'Default')`
- [ ] Default text is in English
- [ ] No hardcoded color classes if this is in group context (use `$color` variable)
- [ ] `wire:model` used appropriately (not `.live` unless needed)
- [ ] No comments in the code
- [ ] Dark mode support on all visual elements (`dark:bg-...`, `dark:text-...`)
- [ ] Primary actions use pink/dynamic color scheme
- [ ] No `alert()`, `confirm()`, or browser dialogs
- [ ] Form validation uses Laravel validation rules
- [ ] Error handling includes `@error` directives or uses `x-fields` components (which include them automatically)

### Step 5: Register Route (if needed)

After creating the component, inform the user if they need to:
1. Add a route in the appropriate routes file (`routes/app.php`, `routes/web.php`, `routes/group.php`)
2. Add navigation links if the component is a full page
3. The component can be embedded in other views with `<livewire:component-name />` or `@livewire('component-name')`

## File Placement

- **CRM components**: `resources/views/livewire/{feature}/{component}.blade.php`
- **Web/public components**: `resources/views/livewire/web/{feature}/{component}.blade.php`
- **Group subdomain components**: `resources/views/livewire/group/{feature}/{component}.blade.php`
- **Shared components**: `resources/views/livewire/{component}.blade.php`
- **Modal components**: `resources/views/livewire/modals/{component}.blade.php`

## Important Reminders

- NEVER create files in `app/Livewire/`. All Livewire logic goes in the Blade file.
- NEVER use `__()` for translations. Always use `text()` or `@text()`.
- If the component interacts with JavaScript (maps, calendars, editors), use `wire:ignore` on the JS-managed sections.
- For file uploads, add `use Livewire\WithFileUploads;` trait and `use WithFileUploads;` in the class.
- For pagination, add `use Livewire\WithPagination;` trait and `use WithPagination;` in the class.
