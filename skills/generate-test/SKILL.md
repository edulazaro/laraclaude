---
name: lc:generate-test
description: Generate feature/unit tests for a model, controller, action, or component by analyzing its code.
argument-hint: "[ModelName | ControllerName | path/to/file]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Generate Test

Generate PHPUnit or Pest test files for a given model, controller, action, or component by analyzing its code and identifying testable behaviors.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Prompt for the class or file to generate tests for. |
| `[ModelName]` | Generate tests for the specified model (searches in `app/Models/`). |
| `[ControllerName]` | Generate tests for the specified controller (searches in `app/Http/Controllers/`). |
| `[path/to/file]` | Generate tests for the file at the given path. |

This is a generator skill -- it always creates files.

## Process

### Step 1: Locate the Target

1. If a path is provided, use `Read` to load the file directly.
2. If a name is provided:
   - Search `app/Models/` for model matches
   - Search `app/Http/Controllers/` for controller matches
   - Search `app/Actions/` for action matches
   - Search `resources/views/livewire/` for Volt component matches
3. If no argument, list common testable locations and ask the user.

### Step 2: Detect Test Framework

1. Check `composer.json` for test dependencies:
   - `pestphp/pest` -> use Pest syntax
   - `phpunit/phpunit` -> use PHPUnit syntax
2. Check for existing tests in `tests/` to match the project's style.
3. Use `Read` to examine 1-2 existing test files for patterns:
   - Import conventions
   - Helper methods used (e.g., `actingAs()`, factory patterns)
   - Assertion styles
   - Database traits (`RefreshDatabase`, `DatabaseTransactions`)

### Step 3: Analyze the Target Class

Use `Read` to load the target file and extract:

#### For Models:
- **Relationships**: Each relationship should have a test verifying it returns the correct type
- **Scopes**: Each scope should have a test verifying the query constraint
- **Accessors/Mutators**: Test the input/output transformation
- **Casts**: Verify cast types work correctly
- **Validation in $fillable**: Test mass assignment protection
- **Observer methods**: Test side effects of creating/updating/deleting
- **Actions**: Test each registered action's behavior
- **Factory**: Check if a factory exists (`database/factories/{Model}Factory.php`)

#### For Controllers:
- **Route methods**: Each public method handling a route
- **Validation rules**: Test that invalid input is rejected
- **Authorization**: Test that unauthorized users are blocked
- **Response types**: Test correct HTTP status codes and response structure
- **Redirects**: Test redirect destinations after actions

#### For Actions:
- **handle() method**: Test with valid and invalid inputs
- **Return values**: Verify expected return types and values
- **Side effects**: Test database changes, events dispatched, jobs queued
- **Edge cases**: Null inputs, empty arrays, boundary values

#### For Volt Components:
- **mount()**: Test initialization with various parameters
- **Public methods**: Test each callable method
- **Validation**: Test form validation rules
- **Wire:model bindings**: Test property updates
- **Events dispatched**: Test `$dispatch()` calls

### Step 4: Generate Test File

Create the test file at the appropriate location:

- **Feature tests**: `tests/Feature/{path matching source}/{ClassName}Test.php`
  - Controllers, Livewire components, full request lifecycle
- **Unit tests**: `tests/Unit/{path matching source}/{ClassName}Test.php`
  - Models, Actions, Services, isolated logic

#### Test Structure:

```php
<?php

namespace Tests\Feature;

use App\Models\Property;
use App\Models\User;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Tests\TestCase;

class PropertyControllerTest extends TestCase
{
    use RefreshDatabase;

    public function test_index_returns_properties_for_authenticated_user(): void
    {
        $user = User::factory()->create();
        $property = Property::factory()->create(['organization_id' => $user->organization_id]);

        $response = $this->actingAs($user)->get(route('properties.index'));

        $response->assertOk();
        $response->assertSee($property->name);
    }

    public function test_store_validates_required_fields(): void
    {
        $user = User::factory()->create();

        $response = $this->actingAs($user)->post(route('properties.store'), []);

        $response->assertSessionHasErrors(['name', 'address']);
    }

    public function test_unauthenticated_user_is_redirected(): void
    {
        $response = $this->get(route('properties.index'));

        $response->assertRedirect(route('login'));
    }
}
```

### Step 5: Include Edge Cases

For each testable method, generate tests for:

1. **Happy path**: Normal expected behavior
2. **Validation failures**: Missing required fields, invalid formats
3. **Authorization**: Wrong user, wrong role, unauthenticated
4. **Not found**: Non-existent IDs, deleted records
5. **Boundary values**: Empty strings, zero, null, max length
6. **Relationships**: Accessing related models, cascading operations

### Step 6: Write the File

1. Use `Write` to create the test file.
2. Ensure the directory structure exists (create if needed).
3. If a factory is needed but does not exist, note it and offer to generate a basic factory.
4. Validate syntax with `php -l` via Bash (Docker-aware).

### Step 7: Report

After generating:

```
TEST FILE GENERATED
====================
File: tests/Feature/Models/PropertyTest.php
Test cases: 12

Tests generated:
  - test_belongs_to_organization
  - test_belongs_to_office
  - test_has_many_operations
  - test_scope_active_filters_correctly
  - test_fillable_fields_are_protected
  - test_soft_delete_works
  - test_accessor_full_address
  - test_toggle_showcase_action
  ...

Run with: docker exec {container} php artisan test --filter=PropertyTest
```

## Notes

- Always check for existing factories before referencing them. If a factory does not exist, use `Model::create()` with explicit attributes instead.
- Use `RefreshDatabase` trait for tests that modify the database.
- For Livewire/Volt components, use `Livewire::test()` syntax.
- For tests requiring authentication, use `actingAs()` with the appropriate guard.
- Match the existing test style in the project (PHPUnit vs Pest, naming conventions).
- Do not generate tests for vendor code or framework internals.
