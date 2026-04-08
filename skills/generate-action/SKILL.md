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

### Step 0: Verify Laractions Is Installed

1. Use `Grep` to check if `edulazaro/laractions` exists in `composer.json`.
2. If NOT found, stop and inform the user:
   ```
   Laractions is not installed. Install it with:
   composer require edulazaro/laractions
   ```
3. If found, continue.

### Step 1: Read the Base Action Class

Read the installed `Action.php` from vendor to understand the current API:

1. Use `Read` to load `vendor/edulazaro/laractions/src/Action.php`.
2. Note all available methods and properties:
   - `create()`, `run()`, `dispatch()`, `handle()`
   - `queue()`, `delay()`, `retry()`
   - `actor()`, `on()`, `with()`
   - `trace()`, `enableLogging()`, `log()`
   - `$rules` for validation
   - `$tries`, `$delay`, `$queue` class properties
3. Also read `vendor/edulazaro/laractions/src/Concerns/HasActions.php` to understand the trait.
4. This ensures the generated code matches the actual installed version, not assumptions.

### Step 2: Parse Arguments

The argument should be in the format `Model/ActionName` (e.g., `Property/ToggleShowcase`, `Client/CreateCopyForGroup`).

- If the argument contains a `/`, split into model name and action name.
- If no `/` is provided, ask the user for both the model name and the action name.
- If no argument at all, ask the user what model and action they need.

Derive the following:
- **Model name**: e.g., `Property`
- **Action class name**: e.g., `ToggleShowcaseAction` (append `Action` suffix if not already present)
- **Action key**: e.g., `toggle_showcase` (snake_case of the action name without the Action suffix)
- **File path**: `app/Actions/{Model}/{ActionClass}.php`

### Step 3: Study Existing Actions

1. Use `Glob` to find existing actions in `app/Actions/{Model}/` directory.
2. If actions exist for this model, use `Read` to examine one or two to understand:
   - Import patterns
   - How the model property is declared
   - Parameter conventions
   - Return type patterns
   - Error handling patterns
   - Whether they use `$rules`, `trace()`, `enableLogging()`, etc.
3. Also read the model file (`app/Models/{Model}.php`) to understand:
   - The `$actions` array (existing action registrations)
   - Available relationships and properties
   - Traits used (check for `HasActions`)

### Step 4: Generate the Action Class

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
- **Match patterns** found in Step 3 for consistency with existing actions.

**IMPORTANT: Generate only the clean skeleton by default.** Do NOT add `$rules`, `$tries`, `$delay`, `$queue`, `trace()`, `enableLogging()`, or `log()` unless the user explicitly asks for them. The generated action should be minimal:

```php
class {ActionClass} extends Action
{
    protected {Model} ${model};

    public function handle(): void
    {
        //
    }
}
```

#### Optional features (only when the user explicitly requests them):

**Validation rules** (user asks for validation):
```php
protected array $rules = [
    'email' => 'required|email',
    'name' => 'required|string|max:255',
];
```

**Queue configuration** (user asks for async/queue support):
```php
protected int $tries = 3;
protected int $delay = 0;
protected string $queue = 'default';
```

**Logging** (user asks for logging):
```php
$this->log('Starting operation', ['id' => $this->{model}->id]);
```

### Step 5: Register the Action in the Model

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

### Step 6: Verify

1. Use `Read` to verify the action class file was created correctly.
2. Use `Read` to verify the model was updated correctly.
3. Run `php -l` on both files to check syntax (Docker-aware: detect container from `docker-compose.yml`).
4. Verify that:
   - The action class extends `EduLazaro\Laractions\Action`
   - The protected model property name matches the lowercase model class name
   - The action key in `$actions` is snake_case
   - The `use` import is present in the model file
   - The `HasActions` trait is used in the model

### Step 7: Show Usage Examples

After successful creation, show how to use the action:

```php
// Synchronous execution
$result = ${model}->action('action_key')->run($param1, $param2);

// With named parameters
$result = ${model}->action('action_key')
    ->with(['param1' => $value1])
    ->run();

// Asynchronous execution (queued)
${model}->action('action_key')
    ->queue('default')
    ->dispatch($param1, $param2);

// With delay and retries
${model}->action('action_key')
    ->queue('default')
    ->delay(60)
    ->retry(3)
    ->dispatch($param1);

// With actor tracking
$user->act(ActionClass::class)
    ->on(${model})
    ->run($param1);

// With audit trail
${model}->action('action_key')
    ->trace()
    ->run($param1);

// With logging enabled
${model}->action('action_key')
    ->enableLogging()
    ->run($param1);
```

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
protected array $rules = [
    'name' => 'required|string',
];

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

### Async/Heavy Action
```php
protected int $tries = 3;
protected int $delay = 0;
protected string $queue = 'default';

public function handle(): void
{
    $this->log('Starting heavy operation', ['id' => $this->{model}->id]);
    // ... heavy logic (API calls, file processing, etc.) ...
    $this->log('Operation completed');
}
```

### Action with Actor Tracking
```php
public function handle(string $reason = ''): void
{
    $this->{model}->update(['status' => 'approved']);
    // Actor is available via $this->actor if set
}
```

## Queueable Actions

Actions can be dispatched to a queue for asynchronous processing:

```php
// Basic queue dispatch
$model->action('action_name')->dispatch($param1);

// Specify queue name
$model->action('action_name')->queue('emails')->dispatch($param1);

// With delay (seconds)
$model->action('action_name')->delay(120)->dispatch($param1);

// With retry attempts
$model->action('action_name')->retry(5)->dispatch($param1);

// Full configuration
$model->action('action_name')
    ->queue('high')
    ->delay(60)
    ->retry(3)
    ->dispatch($param1, $param2);
```

No special code is needed in the action class itself for basic queue support. Laractions wraps the action in an `ActionJob` (implements `ShouldQueue`) automatically.

For actions that should always run on a specific queue, set class-level defaults:

```php
protected int $tries = 3;
protected int $delay = 0;
protected string $queue = 'emails';
```

## Important Notes

- The protected model property name MUST be lowercase version of the model class name (e.g., `protected Property $property`, `protected Client $client`).
- Actions are designed to be reusable and testable units of business logic. Keep them focused on a single responsibility.
- If the action needs to interact with external services, inject them via method parameters or resolve from the container.
- Always check existing actions in the same model directory to maintain consistency.
- Actions support `__invoke()`, so they can be used as callables: `$action($param)` is equivalent to `$action->run($param)`.
- For testing, models with `HasActions` support `mockAction()` to replace actions with mocks.
