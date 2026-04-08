---
name: lc:cache-opportunities
description: Detect repeated queries and computations that should be cached.
argument-hint: "[analyze | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Cache Opportunities

Detect repeated queries, expensive computations, and external API calls that would benefit from caching. Suggests appropriate caching strategies with TTL values.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Scan the entire project for caching opportunities. Read-only report. |
| `fix` | Wrap detected queries/computations in Cache::remember(). Asks for confirmation before each change. |
| `fix --dry-run` | Show what caching code would be added without changing anything. |
| `[file-path]` | Analyze a specific file for caching opportunities. Read-only. |
| `fix [file-path]` | Add caching to a specific file, with confirmation. |

## Detection Rules

### 1. Repeated Configuration/Settings Queries

**What to detect:**
- Settings or configuration loaded from the database on every request
- `Setting::get('key')` or `Config::get()` from database tables called multiple times
- Organization/group settings loaded repeatedly across components

**How to scan:**
1. Use `Grep` to find patterns like `Setting::`, `->settings`, `config(` calls that query the DB.
2. Use `Read` to check if these calls are in frequently-executed code paths (controllers, middleware, Blade views).
3. Check if the same query appears in multiple files without caching.

**Suggested TTL:** 60-300 seconds (settings change infrequently)

**Fix:**
```php
// Before
$settings = Setting::where('organization_id', $orgId)->get();

// After
$settings = Cache::remember("org_settings_{$orgId}", 300, function () use ($orgId) {
    return Setting::where('organization_id', $orgId)->get();
});
```

### 2. Count/Aggregate Queries for Display

**What to detect:**
- `->count()`, `->sum()`, `->avg()` queries used in dashboards or navigation badges
- `withCount()` on large collections for display-only purposes
- Queries that compute totals or statistics shown in the UI

**How to scan:**
1. Use `Grep` to find `->count()`, `->sum()`, `->avg()`, `->max()`, `->min()` calls.
2. Check if they are in dashboard controllers, navigation components, or sidebar widgets.
3. These are often called on every page load but rarely change between requests.

**Suggested TTL:** 60-120 seconds (acceptable staleness for counters)

**Fix:**
```php
// Before
$propertyCount = Property::where('organization_id', $orgId)->count();

// After
$propertyCount = Cache::remember("property_count_{$orgId}", 120, function () use ($orgId) {
    return Property::where('organization_id', $orgId)->count();
});
```

### 3. External API Calls Without Caching

**What to detect:**
- HTTP client calls (`Http::get()`, `Http::post()`, `curl`, `Guzzle`) without surrounding cache
- API responses that are read-only and change infrequently (e.g., geocoding, exchange rates, feed data)
- External service calls made on every request

**How to scan:**
1. Use `Grep` to find `Http::get(`, `Http::post(`, `file_get_contents(http`, and similar patterns.
2. Use `Read` to check if the call is already wrapped in `Cache::remember()`.
3. Identify calls that are read-only (GET requests to external APIs).

**Suggested TTL:** 300-3600 seconds (depending on data freshness needs)

**Fix:**
```php
// Before
$response = Http::get("https://api.example.com/data/{$id}");

// After
$response = Cache::remember("api_data_{$id}", 3600, function () use ($id) {
    return Http::get("https://api.example.com/data/{$id}")->json();
});
```

### 4. Heavy Computations in Blade Views

**What to detect:**
- Complex calculations performed in Blade `@php` blocks or inline PHP
- Collection operations like `->filter()->map()->sort()` chains in views
- Queries executed directly in Blade templates

**How to scan:**
1. Use `Grep` to find `@php` blocks and `<?php` tags in Blade files.
2. Use `Read` to check for query builder calls, complex collection operations, or math computations.
3. Flag any database query in a Blade view (should be in controller/component instead).

**Suggested fix:** Move computation to the controller/component and cache the result there.

### 5. Identical Queries Across Components

**What to detect:**
- The same Eloquent query appearing in multiple Livewire components on the same page
- Shared data (like organization info, user permissions, navigation items) loaded independently in each component

**How to scan:**
1. Use `Grep` to find similar query patterns across multiple Livewire components.
2. Identify queries that produce the same result for the same request context.
3. These could use a shared cache key or be passed as shared data.

**Suggested fix:** Use `Cache::remember()` with a shared cache key, or use Livewire's `#[Computed]` attribute for component-level caching.

## Report Format

```
CACHING OPPORTUNITIES
======================

HIGH IMPACT
------------
1. Dashboard statistics (called on every page load)
   File: app/Http/Controllers/DashboardController.php:25-40
   Queries: 5 aggregate queries (count, sum) on large tables
   Frequency: Every request to /dashboard
   Suggested TTL: 120 seconds
   Estimated savings: ~500ms per request

2. Organization settings (loaded 3x per request)
   Files:
     - app/Http/Middleware/LoadOrganization.php:18
     - resources/views/livewire/layout/navigation.blade.php:8
     - resources/views/livewire/layout/sidebar.blade.php:12
   Suggested TTL: 300 seconds
   Estimated savings: ~100ms per request

MEDIUM IMPACT
--------------
1. External geocoding API call
   File: app/Services/GeocodingService.php:42
   Current: Http::get() on every property save
   Suggested TTL: 86400 seconds (24 hours, addresses rarely change)

LOW IMPACT
-----------
1. User notification count
   File: resources/views/livewire/layout/dropdowns/notifications.blade.php:5
   Note: Already uses polling (4s), adding cache would reduce DB load

SUMMARY
========
High impact opportunities:    3
Medium impact opportunities:  4
Low impact opportunities:     2
Estimated total savings:      ~800ms per request
```

## Fix Mode

For each caching opportunity, wrap the target code in `Cache::remember()`:

1. **Determine cache key**: Use a descriptive key with relevant context variables (e.g., `"org_{$orgId}_property_count"`).
2. **Determine TTL**: Based on data freshness requirements (see suggestions above).
3. **Add cache import**: Ensure `use Illuminate\Support\Facades\Cache;` is imported.
4. **Wrap the code**: Replace the direct query with a `Cache::remember()` call.
5. **Add cache invalidation note**: Comment where cache should be cleared (e.g., in observer when data changes).

**Each change requires explicit user confirmation.**

## Dry Run Mode

Show the before/after code for each proposed caching change, with the cache key and TTL that would be used.

## Notes

- Caching adds complexity. Only cache where the performance benefit justifies the added code.
- Always consider cache invalidation. Suggest where `Cache::forget()` should be called when data changes.
- Redis is preferred over file cache for multi-server deployments. Check `config/cache.php` for the driver.
- Livewire components re-render frequently. Consider `#[Computed]` attribute for component-level caching.
- Do not cache user-specific data with shared keys. Always include the user/organization ID in cache keys.
- Be cautious caching paginated results -- the cache key must include the page number.
