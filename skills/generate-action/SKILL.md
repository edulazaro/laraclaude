---
name: lc:generate-action
description: Create a Laractions action class with proper boilerplate.
argument-hint: "[Model/ActionName]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Generate Laractions Action

Create a Laractions action class following the project's conventions, and register it in the corresponding model.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for model name and action name, then generate. |
| `[Model/ActionName]` | Generate the specified action class and register it in the model. |

This is a generator skill -- it always creates files. There is no analyze-only or fix mode.

## Process

### Step 1: Parse Arguments

The argument should be in the format `Model/ActionName` (e.g., `Property/ToggleShowcase`, `Client/CreateCopyForGroup`).

- If the argument contains a `/`, split into model name and action name.
- If no `/` is provided, ask the user for both the model name and the action name.
- If no argument at all, ask the user what model and action they need.

Derive the following:
- **Model name**: e.g., `Property`
- **Action class name**: e.g., `ToggleShowcaseAction` (append `Action` suffix if not already present)
- **Action key**: e.g., `toggle_showcase` (snake_case of the action name without the Action suffix)
- **File path**: `app/Actions/{Model}/{ActionClass}.php`

### Step 2: Study Existing Actions

1. Use `Glob` to find existing actions in `app/Actions/{Model}/` directory.
2. If actions exist for this model, use `Read` to examine one or two to understand:
   - Import patterns
   - How the model property is declared
   - Parameter conventions
   - Return type patterns
   - Error handling patterns
3. Also read the model file (`app/Models/{Model}.php`) to understand:
   - The `$actions` array (existing action registrations)
   - Available relationships and properties
   - Traits used

### Step 3: Generate the Action Class

Create the file using `Write` at `app/Actions/{Model}/{ActionClass}.php`:

```php
<?php

namespace App\Actions\{Model};

use App\Models\{Model};
use EduLazaro\Laractions\Action;

class {ActionClass} extends Action
{
    protected {Model} ${model};

    public function handle(): void
    {
        // Action logic here
    }
}
```

Key requirements:
- **Namespace**: `App\Actions\{Model}`
- **Extends**: `EduLazaro\Laractions\Action`
- **Protected model property**: Must match the model class name in lowercase (e.g., `protected Property $property`)
- **handle() method**: Contains the action's business logic. Parameters and return type vary by action purpose.
- **No comments** unless the user explicitly requests them.
- **Use model imports** at the top with `use` statements, never inline qualified class names.
- **Use `text()` helper** for any translatable strings (Laratext, not `__()`)

### Step 4: Register the Action in the Model

1. Use `Read` to load the model file at `app/Models/{Model}.php`.
2. Find the `$actions` array property.
3. Use `Edit` to add the new action registration:

```php
protected array $actions = [
    // ... existing actions
    'action_key' => ActionClass::class,
];
```

4. Add the `use` import for the new action class at the top of the model file if not already present.

If the model does not have:
- `use EduLazaro\Laractions\Concerns\HasActions;` trait -- add it
- `$actions` array property -- create it

### Step 5: Verify

1. Check that the directory `app/Actions/{Model}/` exists. If not, it will be created when writing the file.
2. Verify the action class file was created correctly using `Read`.
3. Verify the model was updated correctly using `Read`.

## Action Patterns

Based on common patterns in the project, here are templates for different action types:

### Data Manipulation Action
```php
public function handle(array $data): void
{
    $this->{model}->update($data);
}
```

### Copy/Clone Action
```php
public function handle(Organization|int|string $organization, int|string|null $officeId = null): {Model}
{
    $orgInstance = $organization instanceof Organization ? $organization : Organization::find($organization);

    return {Model}::create([
        'organization_id' => $orgInstance->id,
        'office_id' => $officeId,
        // ... copy fields from $this->{model}
    ]);
}
```

### Toggle Action
```php
public function handle(): void
{
    $this->{model}->update([
        'active' => !$this->{model}->active,
    ]);
}
```

### Validation/Check Action
```php
public function handle(): array
{
    $errors = [];

    if (!$this->{model}->name) {
        $errors[] = text('name_required', 'Name is required');
    }

    return $errors;
}
```

### Activity Recording Action
```php
public function handle(array $oldValues, array $newValues): void
{
    foreach ($newValues as $field => $newValue) {
        $oldValue = $oldValues[$field] ?? null;
        if ($oldValue !== $newValue) {
            Activity::create([
                'activitable_type' => '{model}',
                'activitable_id' => $this->{model}->id,
                'field' => $field,
                'old_value' => $oldValue,
                'new_value' => $newValue,
            ]);
        }
    }
}
```

### Queueable Action
Some actions can be dispatched to a queue:
```php
// Usage:
$model->action('action_name')->queue('default')->dispatch();
```
No special code needed in the action class itself -- Laractions handles the queueing.

## Usage Examples

After creation, the action can be called:

```php
// Direct run with parameters
$result = $model->action('action_key')->run($param1, $param2);

// Using with() to set parameters
$result = $model->action('action_key')
    ->with(['param1' => $value1])
    ->run();

// Queue for background processing
$model->action('action_key')->queue('default')->dispatch();
```

## Important Notes

- The protected model property name MUST be lowercase version of the model class name (e.g., `protected Property $property`, `protected Client $client`).
- Actions are designed to be reusable and testable units of business logic. Keep them focused on a single responsibility.
- If the action needs to interact with external services, inject them via method parameters or resolve from the container.
- Always check existing actions in the same model directory to maintain consistency.
