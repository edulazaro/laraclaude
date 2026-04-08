---
name: lc:extract-action
description: Extract business logic from a controller or Livewire component method into a Laractions Action class.
argument-hint: "[file:method]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Extract Action

Extract business logic from a controller method or Livewire component method into a standalone Laractions Action class. Replaces the original code with an action call.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for the file path and method name to extract from. |
| `[file:method]` | Extract the specified method's logic into an action. Format: `app/Http/Controllers/PropertyController.php:store` or `resources/views/livewire/properties/edit.blade.php:save`. |

This is a refactoring skill -- it creates an action file and modifies the source file.

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

### Step 2: Locate the Source Method

1. Parse the argument to extract file path and method name (separated by `:`).
2. If no argument provided, ask the user for the file and method.
3. Use `Read` to load the source file.
4. Find the target method and extract its full body.

### Step 3: Identify the Model

Analyze the method body to determine which model the action operates on:

1. Look for Eloquent model usage: `$property->update()`, `Property::create()`, `$this->property->save()`
2. Check controller constructor injection or method parameters for model type hints
3. For Volt components, check public properties for model instances
4. If multiple models are involved, identify the primary model (the one being acted upon)

### Step 4: Study Existing Actions

1. Use `Glob` to find existing actions in `app/Actions/{Model}/` directory.
2. If actions exist, use `Read` to examine one or two to understand conventions:
   - Import patterns, parameter passing, return types, error handling
   - Whether they use `$rules`, `trace()`, `enableLogging()`, queue properties, etc.
3. Match the new action to these patterns for consistency.

### Step 5: Separate Concerns

Analyze the method and separate:

#### Business Logic (goes into the Action):
- Data validation and transformation
- Database operations (create, update, delete)
- Relationship management (sync, attach, detach)
- Calculations and computations
- Side effects (dispatching events, sending notifications)
- External service calls

#### HTTP/UI Concerns (stays in the controller/component):
- Request input extraction (`$request->input()`, `$this->validate()`)
- Authorization checks (`$this->authorize()`)
- Response building (`return redirect()`, `return view()`)
- Flash messages (`session()->flash()`)
- Livewire dispatching (`$this->dispatch()`)

### Step 6: Design the Action

Determine the action's interface:

1. **Action name**: Derive from the method name and context (e.g., `store` method on PropertyController -> `CreatePropertyAction`)
2. **Parameters**: What data the action needs (extracted from the method's dependencies)
3. **Return type**: What the action returns (the created/updated model, a status, void)
4. **Model property**: The primary model instance the action operates on
5. **Validation rules**: If the extracted logic validates data, move those rules to `$rules`
6. **Queue suitability**: If the logic is heavy (API calls, file processing, email sending), suggest async execution

### Step 7: Generate the Action Class

Create the action file at `app/Actions/{Model}/{ActionName}.php`:

```php
<?php

namespace App\Actions\Property;

use App\Models\Property;
use EduLazaro\Laractions\Action;

class CreatePropertyAction extends Action
{
    protected Property $property;

    public function handle(array $data): Property
    {
        $this->property->update([
            'name' => $data['name'],
            'address' => $data['address'],
            'price' => $data['price'],
        ]);

        if (isset($data['tags'])) {
            $this->property->tags()->sync($data['tags']);
        }

        return $this->property->fresh();
    }
}
```

Use `Write` to create the file.

**IMPORTANT: Do NOT add `$rules`, `$tries`, `$delay`, `$queue`, `trace()`, `enableLogging()`, or `log()` unless the user explicitly asks for them.** Extract only the business logic into a clean action. These features are available but should only be added on request.

### Step 8: Register in Model

1. Use `Read` to load the model file.
2. Add the action to the `$actions` array using `Edit`:
   ```php
   'create_property' => CreatePropertyAction::class,
   ```
3. Add the `use` import for the action class.
4. If the model lacks `HasActions` trait, add it:
   - Add `use EduLazaro\Laractions\Concerns\HasActions;` import
   - Add `use HasActions;` in the class body

### Step 9: Refactor the Original Method

Use `Edit` to replace the business logic in the original method with the action call.

**Before (controller):**
```php
public function store(Request $request, Property $property)
{
    $validated = $request->validate([...]);

    $property->update([
        'name' => $validated['name'],
        'address' => $validated['address'],
    ]);

    if (isset($validated['tags'])) {
        $property->tags()->sync($validated['tags']);
    }

    return redirect()->route('properties.show', $property)
        ->with('success', text('property_updated', 'Property updated'));
}
```

**After (controller) -- synchronous:**
```php
public function store(Request $request, Property $property)
{
    $validated = $request->validate([...]);

    $property->action('create_property')->run($validated);

    return redirect()->route('properties.show', $property)
        ->with('success', text('property_updated', 'Property updated'));
}
```

**After (controller) -- asynchronous (for heavy operations):**
```php
public function store(Request $request, Property $property)
{
    $validated = $request->validate([...]);

    $property->action('create_property')
        ->queue('default')
        ->dispatch($validated);

    return redirect()->route('properties.show', $property)
        ->with('success', text('property_queued', 'Property update queued'));
}
```

**Before (Volt component):**
```php
public function save()
{
    $this->validate();

    $this->property->update([
        'name' => $this->name,
        'address' => $this->address,
    ]);

    session()->flash('success', text('saved', 'Saved'));
}
```

**After (Volt component):**
```php
public function save()
{
    $this->validate();

    $this->property->action('update_property')->run([
        'name' => $this->name,
        'address' => $this->address,
    ]);

    session()->flash('success', text('saved', 'Saved'));
}
```

### Step 10: Verify

1. Use `Read` to verify the action class, model, and source file are all correct.
2. Run `php -l` on all modified files to check syntax (Docker-aware: detect container from `docker-compose.yml`).
3. Verify that:
   - The action class extends `EduLazaro\Laractions\Action`
   - The protected model property name matches the lowercase model class name
   - The action key in `$actions` is snake_case
   - The `use` import is present in the model file
   - The `HasActions` trait is used in the model
   - The original method now calls the action instead of containing business logic
   - HTTP/UI concerns remain in the controller/component
4. Show a summary of changes:
   - Action class created at: `{path}`
   - Model updated: `{path}` (action registered)
   - Source refactored: `{path}` (business logic replaced with action call)

### Step 11: Show Usage Examples

Show the user how to use the extracted action in different contexts:

```php
// Synchronous (default)
${model}->action('action_key')->run($data);

// With named parameters
${model}->action('action_key')
    ->with(['key' => $value])
    ->run();

// Asynchronous (queued)
${model}->action('action_key')
    ->queue('default')
    ->dispatch($data);

// With delay and retries
${model}->action('action_key')
    ->queue('default')
    ->delay(60)
    ->retry(3)
    ->dispatch($data);

// With actor tracking
auth()->user()->act(ActionClass::class)
    ->on(${model})
    ->trace()
    ->run($data);

// As callable
$action = ${model}->action('action_key');
$action($data); // Same as ->run($data)
```

## Guidelines

- **Keep the action focused**: One action, one responsibility. If the method does multiple unrelated things, suggest splitting into multiple actions.
- **Keep validation in the caller**: The controller/component validates input. The action receives clean data. However, if validation is part of the business logic itself, use `$rules` in the action.
- **Keep UI responses in the caller**: Flash messages, redirects, and event dispatching stay in the controller/component.
- **Match existing patterns**: Read other actions for the same model to maintain consistency in parameter passing, return types, and error handling.
- **Use text() for translations**: Any translatable strings in the action should use the `text()` helper.
- **No comments**: Unless the user explicitly requests them.
- **Suggest async when appropriate**: If the extracted logic involves external API calls, file processing, email sending, or heavy computations, suggest using `->queue()->dispatch()` instead of `->run()`.
- **Suggest tracing when appropriate**: If the action modifies critical data or performs irreversible operations, suggest using `->trace()` for audit trail.

## Notes

- This skill creates a new action class AND modifies the source file. Both changes happen with user confirmation.
- If the method is too simple (one or two lines of logic), advise against extraction -- the overhead of an action class may not be justified.
- If the method has complex conditional logic that depends on HTTP context, suggest which parts to extract and which to leave.
- Always add the `use` import for the action class in the model file.
- Actions support `__invoke()`, so they can be used as callables.
- For testing, models with `HasActions` support `mockAction()` to replace actions with mocks.
