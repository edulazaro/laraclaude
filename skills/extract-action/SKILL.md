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

### Step 1: Locate the Source Method

1. Parse the argument to extract file path and method name (separated by `:`).
2. If no argument provided, ask the user for the file and method.
3. Use `Read` to load the source file.
4. Find the target method and extract its full body.

### Step 2: Identify the Model

Analyze the method body to determine which model the action operates on:

1. Look for Eloquent model usage: `$property->update()`, `Property::create()`, `$this->property->save()`
2. Check controller constructor injection or method parameters for model type hints
3. For Volt components, check public properties for model instances
4. If multiple models are involved, identify the primary model (the one being acted upon)

### Step 3: Separate Concerns

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

### Step 4: Design the Action

Determine the action's interface:

1. **Action name**: Derive from the method name and context (e.g., `store` method on PropertyController -> `CreatePropertyAction`)
2. **Parameters**: What data the action needs (extracted from the method's dependencies)
3. **Return type**: What the action returns (the created/updated model, a status, void)
4. **Model property**: The primary model instance the action operates on

### Step 5: Generate the Action Class

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
        // Business logic extracted from controller
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

### Step 6: Register in Model

1. Use `Read` to load the model file.
2. Add the action to the `$actions` array using `Edit`:
   ```php
   'create_property' => CreatePropertyAction::class,
   ```
3. Add the `use` import for the action class.
4. If the model lacks `HasActions` trait, add it.

### Step 7: Refactor the Original Method

Use `Edit` to replace the business logic in the original method with the action call:

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

**After (controller):**
```php
public function store(Request $request, Property $property)
{
    $validated = $request->validate([...]);

    $property->action('create_property')->run($validated);

    return redirect()->route('properties.show', $property)
        ->with('success', text('property_updated', 'Property updated'));
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

### Step 8: Verify

1. Check that all files are syntactically valid using `php -l` (Docker-aware).
2. Show a summary of changes made:
   - Action class created at: `{path}`
   - Model updated: `{path}` (action registered)
   - Source refactored: `{path}` (business logic replaced with action call)

## Guidelines

- **Keep the action focused**: One action, one responsibility. If the method does multiple unrelated things, suggest splitting into multiple actions.
- **Keep validation in the caller**: The controller/component validates input. The action receives clean data.
- **Keep UI responses in the caller**: Flash messages, redirects, and event dispatching stay in the controller/component.
- **Match existing patterns**: Read other actions for the same model to maintain consistency in parameter passing, return types, and error handling.
- **Use text() for translations**: Any translatable strings in the action should use the `text()` helper.
- **No comments**: Unless the user explicitly requests them.

## Notes

- This skill creates a new action class AND modifies the source file. Both changes happen with user confirmation.
- If the method is too simple (one or two lines of logic), advise against extraction -- the overhead of an action class may not be justified.
- If the method has complex conditional logic that depends on HTTP context, suggest which parts to extract and which to leave.
- Always add the `use` import for the action class in the model file.
