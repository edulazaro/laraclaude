---
name: lc:generate-modal
description: Generate a modal component using the project's x-modal pattern with Livewire integration.
argument-hint: "[ModalName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob Agent
---

## Subcommands

| Subcommand | Description |
|---|---|
| `<ModalName>` | Generate a modal Volt component with the given name (e.g., `EditUser`, `ConfirmDelete`, `CreateInvoice`) |

## Overview

This is a generator skill. It creates a complete modal Volt component following the project's established patterns: `<x-modal>` component, Livewire/Volt single-file architecture, proper event dispatching, and form handling with validation.

## Instructions

### Step 1: Parse the argument and determine file path

Convert the ModalName to a blade file path:

- `EditUser` -> `resources/views/livewire/modals/edit-user.blade.php`
- `CreateInvoice` -> `resources/views/livewire/modals/create-invoice.blade.php`
- `ConfirmDelete` -> `resources/views/livewire/modals/confirm-delete.blade.php`

Convert PascalCase to kebab-case for the filename.

The modal ID will be derived from the filename: `edit-user`, `create-invoice`, `confirm-delete`.

### Step 2: Verify prerequisites

1. Check that the target directory exists: `resources/views/livewire/modals/`
2. Check that the file does not already exist (warn and abort if it does)
3. Read the `<x-modal>` component to understand available attributes:
   - Read `resources/views/components/modal.blade.php` or `app/View/Components/Modal.php`
   - Confirm valid sizes: `sm, md, lg, xl, 2xl, 3xl, 7xl, 8xl`
4. Scan existing modal files for patterns:
   - Use Glob: `resources/views/livewire/modals/*.blade.php`
   - Read 1-2 existing modals to match the project's exact conventions

### Step 3: Ask the user for modal details (if not provided)

If the user only provided a name, ask:
- What fields/form inputs does the modal need?
- What model does it interact with (if any)?
- What size? (default: `2xl`)
- Is this a confirmation modal (simple yes/no) or a form modal?

If the context makes the purpose clear, infer the answers and proceed.

### Step 4: Generate the Volt component

Create the file at the determined path using the Write tool.

**Template structure**:

```php
<?php

use App\Models\ModelName;
use Livewire\Volt\Component;
use Livewire\Attributes\On;

new class extends Component {
    public ?int $modelId = null;
    public string $fieldName = '';
    // ... other form fields as public properties

    #[On('open-modal-name')]
    public function loadData(int $id): void
    {
        $model = ModelName::findOrFail($id);
        $this->modelId = $model->id;
        $this->fieldName = $model->field_name;
        // ... populate form fields

        $this->dispatch('open-modal', id: 'modal-id');
    }

    public function submit(): void
    {
        $this->validate([
            'fieldName' => 'required|string|max:255',
            // ... validation rules
        ]);

        $model = ModelName::findOrFail($this->modelId);
        $model->update([
            'field_name' => $this->fieldName,
            // ... other fields
        ]);

        $this->dispatch('close-modal');
        $this->dispatch('model-updated');
        $this->reset();
    }

    public function resetForm(): void
    {
        $this->reset();
        $this->resetValidation();
    }
}; ?>

<x-modal id="modal-id" :title="text('modal_title', 'Modal Title')" size="2xl">
    <div class="space-y-4">
        <div>
            <x-fields.text-input
                wire:model="fieldName"
                :label="text('field_name', 'Field Name')"
                id="fieldName"
            />
        </div>

        {{-- Add more fields as needed --}}
    </div>

    <x-slot name="footer">
        <button
            type="button"
            @click="$dispatch('close-modal')"
            class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-lg hover:bg-gray-50 dark:bg-gray-700 dark:text-gray-300 dark:border-gray-600 dark:hover:bg-gray-600"
        >
            @text('cancel', 'Cancel')
        </button>
        <button
            type="submit"
            class="px-4 py-2 text-sm font-medium text-white bg-pink-600 rounded-lg hover:bg-pink-700 focus:ring-4 focus:ring-pink-300 dark:bg-pink-600 dark:hover:bg-pink-700 dark:focus:ring-pink-800"
        >
            @text('save', 'Save')
        </button>
    </x-slot>
</x-modal>
```

### Step 5: Key conventions to follow

1. **ALWAYS use Volt single-file pattern** - PHP logic at top of blade file, never separate class files
2. **Use `@text()` for ALL user-visible text** - Never hardcode strings. Default text must be in English
3. **Use `x-fields.*` components for form inputs** - These already include `@error` directives. Do NOT add duplicate `<x-input-error>` components
4. **Modal opens via event dispatch** - Use `$dispatch('open-modal', { id: 'modal-id' })` from parent
5. **Modal closes via dispatch** - Use `$dispatch('close-modal')` or `$this->dispatch('close-modal')` from PHP
6. **Form submits via wire:submit** - The `<x-modal>` component already wraps content in a form with `wire:submit="submit"`
7. **Reset form on close** - Call `$this->reset()` and `$this->resetValidation()` after successful submit
8. **Emit events for parent** - After successful actions, dispatch events so parent components can refresh (e.g., `$this->dispatch('model-updated')`)
9. **Model imports** - Always use `use App\Models\ModelName;` at the top, never inline `\App\Models\ModelName`
10. **No comments** - Unless explicitly requested by the user

### Step 6: Confirmation modal variant

For confirmation/delete modals, use a simpler pattern:

```php
<?php

use App\Models\ModelName;
use Livewire\Volt\Component;
use Livewire\Attributes\On;

new class extends Component {
    public ?int $modelId = null;
    public string $modelName = '';

    #[On('confirm-delete-model')]
    public function setModel(int $id): void
    {
        $model = ModelName::findOrFail($id);
        $this->modelId = $model->id;
        $this->modelName = $model->name;

        $this->dispatch('open-modal', id: 'confirm-delete-model');
    }

    public function submit(): void
    {
        $model = ModelName::findOrFail($this->modelId);
        $model->delete();

        $this->dispatch('close-modal');
        $this->dispatch('model-deleted');
        $this->reset();
    }
}; ?>

<x-modal id="confirm-delete-model" :title="text('confirm_deletion', 'Confirm Deletion')" size="md">
    <p class="text-gray-600 dark:text-gray-400">
        @text('confirm_delete_message', 'Are you sure you want to delete')
        <strong>{{ $modelName }}</strong>?
    </p>

    <x-slot name="footer">
        <button
            type="button"
            @click="$dispatch('close-modal')"
            class="px-4 py-2 text-sm font-medium text-gray-700 bg-white border border-gray-300 rounded-lg hover:bg-gray-50 dark:bg-gray-700 dark:text-gray-300 dark:border-gray-600 dark:hover:bg-gray-600"
        >
            @text('cancel', 'Cancel')
        </button>
        <button
            type="submit"
            class="px-4 py-2 text-sm font-medium text-white bg-red-600 rounded-lg hover:bg-red-700 focus:ring-4 focus:ring-red-300 dark:bg-red-600 dark:hover:bg-red-700 dark:focus:ring-red-800"
        >
            @text('delete', 'Delete')
        </button>
    </x-slot>
</x-modal>
```

### Step 7: Show usage instructions

After generating the file, show the user how to use the modal:

```
## Generated: resources/views/livewire/modals/edit-user.blade.php

### Include in parent view:
<livewire:modals.edit-user />

### Open from parent component (Blade/Alpine):
<button @click="$dispatch('open-edit-user', { id: {{ $user->id }} })">
    Edit
</button>

### Open from parent component (Livewire PHP):
$this->dispatch('open-edit-user', id: $userId);

### Listen for updates in parent:
#[On('model-updated')]
public function refreshData(): void
{
    // Refresh your data
}
```
