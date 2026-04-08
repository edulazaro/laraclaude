---
name: lc:api-docs
description: Generate API documentation from routes, controllers, and form requests.
argument-hint: "[all | route-prefix]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# API Documentation Generator

Generate API documentation by analyzing route definitions, controller methods, form request validation rules, and response structures.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Generate documentation for all API routes. |
| `all` | Same as no argument -- document all API routes. |
| `[route-prefix]` | Document only routes matching the given prefix (e.g., `api/v1/properties`). |

This is a generator skill -- it produces documentation output.

## Process

### Step 1: Collect API Routes

1. Use `Read` to load `routes/api.php` and any included route files.
2. Use `Bash` to get the full route list from Laravel (if Docker is available):
   ```
   docker exec {container} php artisan route:list --json --path=api
   ```
3. If Docker is unavailable, parse route files manually to extract:
   - HTTP method (GET, POST, PUT, PATCH, DELETE)
   - URI pattern (e.g., `/api/v1/properties/{property}`)
   - Route name
   - Controller and method
   - Middleware applied

4. If a `[route-prefix]` argument is provided, filter routes to only those matching the prefix.

### Step 2: Analyze Each Endpoint

For each API route, use `Read` to load the controller and extract:

#### A. Request Details
1. **URL parameters**: Extract from route definition (`{property}`, `{id}`)
2. **Query parameters**: Look for `$request->query()`, `$request->input()` in GET handlers
3. **Request body**: Look for:
   - Form Request class (in method signature) -> read its `rules()` method
   - `$request->validate([...])` inline validation
   - `$request->input('field')` or `$request->only([...])` for expected fields
4. **Headers**: Check for custom header requirements in middleware or controller

#### B. Validation Rules
For each validated field, extract:
- Field name
- Rules (required, string, max:255, etc.)
- Type derived from rules (string, integer, boolean, array, file)
- Whether the field is required or optional

#### C. Response Structure
Analyze the controller method's return statements:
- `return response()->json($data)` -> extract `$data` structure
- `return new JsonResource($model)` -> read the resource class's `toArray()` method
- `return Model::collection($models)` -> read the resource collection
- HTTP status codes used (`200`, `201`, `204`, `404`, `422`)

#### D. Authentication & Authorization
- Middleware: `auth:sanctum`, `auth:api`, or custom auth middleware
- Policy checks: `$this->authorize()`, `Gate::allows()`
- Scoped to specific user/organization

### Step 3: Generate Documentation

Output structured markdown documentation:

```markdown
# API Documentation

## Authentication

All API endpoints require authentication via Bearer token (Laravel Sanctum).

Include the token in the `Authorization` header:
```
Authorization: Bearer {your-api-token}
```

---

## Properties

### List Properties

`GET /api/v1/properties`

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| page | integer | No | Page number for pagination |
| per_page | integer | No | Items per page (default: 15) |
| search | string | No | Search by name or reference |
| status | string | No | Filter by status (active, draft, archived) |

**Response (200):**

```json
{
    "data": [
        {
            "id": 1,
            "name": "Beach Villa",
            "reference": "AB0012",
            "status": "active",
            "price": 450000,
            "created_at": "2026-01-15T10:30:00Z"
        }
    ],
    "meta": {
        "current_page": 1,
        "last_page": 5,
        "per_page": 15,
        "total": 72
    }
}
```

---

### Create Property

`POST /api/v1/properties`

**Request Body:**

| Field | Type | Required | Rules |
|-------|------|----------|-------|
| name | string | Yes | max:255 |
| address | string | Yes | max:500 |
| price | decimal | Yes | min:0 |
| status | string | No | in:active,draft |

**Response (201):**

```json
{
    "data": {
        "id": 73,
        "name": "New Property",
        "reference": "AB0073",
        "status": "draft"
    }
}
```

**Error Response (422):**

```json
{
    "message": "The given data was invalid.",
    "errors": {
        "name": ["The name field is required."],
        "price": ["The price must be at least 0."]
    }
}
```
```

### Step 4: Include Example Requests

For each endpoint, generate example curl commands:

```bash
# List properties
curl -X GET "https://api.example.com/api/v1/properties?page=1&status=active" \
  -H "Authorization: Bearer {token}" \
  -H "Accept: application/json"

# Create property
curl -X POST "https://api.example.com/api/v1/properties" \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{"name": "Beach Villa", "address": "123 Coast Rd", "price": 450000}'
```

### Step 5: Output

Present the full documentation as formatted markdown in the response. If the user wants to save it to a file, offer to write it to a specified location (e.g., `docs/api.md`), but do not create files unless asked.

## Notes

- This generates documentation from code analysis, not runtime inspection.
- Response structures are inferred from the code. Actual responses may vary.
- If API Resources (`JsonResource`) are used, read the `toArray()` method for accurate response structures.
- For Eloquent API Resources, also check `$hidden` and `$visible` on the model.
- Polymorphic responses (different structures based on conditions) are noted but may not be fully documented.
- Webhook endpoints may have different authentication (e.g., signature verification instead of Bearer tokens).
