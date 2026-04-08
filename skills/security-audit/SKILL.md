---
name: lc:security-audit
description: Detect common Laravel security vulnerabilities - SQL injection, XSS, mass assignment, exposed secrets.
argument-hint: "[analyze | fix | fix --dry-run | file-path | fix file-path]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Security Audit

Detect common security vulnerabilities in a Laravel application through static analysis. Checks for SQL injection risks, XSS vectors, mass assignment issues, exposed secrets, missing CSRF, and debug mode misconfiguration.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Scan the entire project and report all security issues. Read-only. |
| `fix` | Scan and auto-fix all fixable issues. Asks for confirmation before each change. |
| `fix --dry-run` | Show what fixes would be applied without changing anything. |
| `[file-path]` | Audit a specific file only. Read-only. |
| `fix [file-path]` | Fix security issues in a specific file, with confirmation. |

## Detection Rules

### 1. SQL Injection via DB::raw()

**What to detect:**
- `DB::raw()` calls containing variable interpolation or concatenation with user input
- `whereRaw()`, `selectRaw()`, `orderByRaw()`, `havingRaw()` with unparameterized variables
- `DB::statement()` or `DB::unprepared()` with string concatenation

**How to scan:**
1. Use `Grep` to find all `DB::raw(`, `whereRaw(`, `selectRaw(`, `orderByRaw(`, `havingRaw(`, `DB::statement(`, `DB::unprepared(` calls.
2. Use `Read` to examine each match and check if the SQL string contains `$` variables without being passed as bindings.

**Safe patterns (do not flag):**
```php
DB::raw('COUNT(*)')                          // No variables
whereRaw('price > ?', [$minPrice])           // Parameterized
DB::select('SELECT * FROM t WHERE id = ?', [$id])  // Parameterized
```

**Unsafe patterns (flag):**
```php
DB::raw("price > $minPrice")                // Variable interpolation
whereRaw("name LIKE '%{$search}%'")         // Variable in string
DB::statement("ALTER TABLE $table ...")      // Variable table name
```

**Severity:** Critical

**Fix:** Replace with parameterized queries using `?` placeholders and binding arrays.

### 2. XSS via Unescaped Blade Output

**What to detect:**
- `{!! $variable !!}` in Blade templates where `$variable` could contain user input
- `{!! !!}` is safe when rendering trusted HTML (e.g., rich text from admin, markdown output)

**How to scan:**
1. Use `Grep` to find all `{!!` patterns in `resources/views/**/*.blade.php`.
2. Use `Read` to check the context of each match.
3. Flag instances where the variable comes from user input (form data, request parameters, database fields editable by users).

**Safe patterns (do not flag):**
```blade
{!! $markdownHtml !!}          <!-- Admin-generated content -->
{!! $svgIcon !!}               <!-- Static SVG -->
{!! $component->render() !!}   <!-- Component output -->
```

**Unsafe patterns (flag):**
```blade
{!! $user->bio !!}             <!-- User-editable content -->
{!! $comment->body !!}         <!-- User-submitted content -->
{!! request('content') !!}     <!-- Direct request input -->
```

**Severity:** High

**Fix:** Replace `{!! !!}` with `{{ }}` for auto-escaping, or apply `e()` / `htmlspecialchars()` / `Str::of()->sanitizeHtml()`.

### 3. Models Without Mass Assignment Protection

**What to detect:**
- Models that define neither `$fillable` nor `$guarded`
- Models with `$guarded = []` (completely unguarded)

**How to scan:**
1. Use `Glob` to find all `app/Models/*.php`.
2. Use `Read` to check each model for `$fillable` or `$guarded` properties.
3. Flag models missing both, or with empty `$guarded`.

**Severity:** High

**Fix:** Add appropriate `$fillable` array based on the migration columns (exclude id, timestamps, auto-generated fields).

### 4. Hardcoded Secrets in Code

**What to detect:**
- Passwords, API keys, tokens hardcoded in PHP files (not using `env()`)
- Patterns: `'password' => 'actual_value'`, `$apiKey = 'sk-...'`, `$secret = '...'`
- Database credentials in config files not using `env()`

**How to scan:**
1. Use `Grep` to search for common secret patterns:
   - `password.*=.*['"]` (excluding `env(` on the same line)
   - `secret.*=.*['"]` (excluding `env(`)
   - `api_key.*=.*['"]` (excluding `env(`)
   - `token.*=.*['"]` (excluding `env(`)
   - Strings starting with `sk-`, `pk-`, `whsec_`, `Bearer ` that are not in `.env`
2. Use `Read` to verify each match is a real hardcoded secret and not a variable name, comment, or config key.

**Severity:** Critical

**Fix:** Move the value to `.env` and replace with `env('KEY_NAME')` or `config('key')`.

### 5. Real Credentials in .env.example

**What to detect:**
- `.env.example` containing values that look like real credentials instead of placeholder values
- Patterns: long random strings, actual database passwords, API keys with valid formats

**How to scan:**
1. Use `Read` to load `.env.example`.
2. Check each value against patterns:
   - DB_PASSWORD with a non-empty value that is not `password`, `secret`, `''`, or similar placeholder
   - API keys matching real formats (e.g., `sk_live_...`, `pk_live_...`)
   - Long base64 or hex strings that are not example/placeholder values

**Severity:** High

**Fix:** Replace real values with placeholder text (e.g., `your-api-key-here`, empty string, or `secret`).

### 6. Missing CSRF Protection

**What to detect:**
- POST/PUT/PATCH/DELETE routes without CSRF middleware
- Forms in Blade without `@csrf` directive
- Routes excluded from CSRF verification in `VerifyCsrfToken` middleware

**How to scan:**
1. Use `Grep` to find POST/PUT/PATCH/DELETE route definitions in `routes/*.php`.
2. Check if they have the `web` middleware group (which includes CSRF) or explicit `csrf` middleware.
3. Use `Grep` to find `<form` tags in Blade files and check for `@csrf`.
4. Check `app/Http/Middleware/VerifyCsrfToken.php` for `$except` exclusions.

**Severity:** Medium (API routes are exempt, web routes need it)

**Fix:** Add `@csrf` to forms, ensure POST routes use `web` middleware group.

### 7. Debug Mode and Error Exposure

**What to detect:**
- `APP_DEBUG=true` in production-related config
- `'debug' => true` hardcoded in `config/app.php` (not using `env()`)
- Detailed error pages exposed in production

**How to scan:**
1. Use `Read` to check `config/app.php` for the `debug` key.
2. Check if it uses `env('APP_DEBUG', false)` (safe) or a hardcoded value (unsafe).
3. Check `.env` and `.env.example` for `APP_DEBUG` value.

**Severity:** Medium

**Fix:** Ensure `config/app.php` uses `env('APP_DEBUG', false)`.

## Report Format

```
SECURITY AUDIT REPORT
======================

CRITICAL (2 issues)
--------------------
1. SQL Injection Risk
   File: app/Services/SearchService.php:45
   Code: DB::raw("MATCH(title) AGAINST('$query')")
   Fix: Use parameterized query: DB::raw("MATCH(title) AGAINST(?)", [$query])

2. Hardcoded API Key
   File: config/services.php:28
   Code: 'secret' => 'sk_live_abc123...'
   Fix: Replace with env('STRIPE_SECRET')

HIGH (3 issues)
----------------
1. Mass Assignment Unprotected
   File: app/Models/Payment.php
   Issue: No $fillable or $guarded defined
   Fix: Add $fillable with appropriate columns

...

SUMMARY
========
Files scanned:    245
Critical issues:  2
High issues:      3
Medium issues:    5
Low issues:       1
```

## Fix Mode

For each fixable issue, apply the fix with confirmation:

1. **Mass assignment**: Add `$fillable` array based on migration columns.
2. **XSS**: Replace `{!! !!}` with `{{ }}` where appropriate.
3. **Hardcoded secrets**: Replace with `env()` calls and add placeholder to `.env.example`.
4. **Missing CSRF**: Add `@csrf` to forms.
5. **Debug mode**: Update config to use `env('APP_DEBUG', false)`.

SQL injection fixes require manual review and are flagged but not auto-fixed (risk of breaking queries).

## Notes

- This is static analysis only. It cannot detect runtime vulnerabilities or configuration issues on the actual server.
- False positives will occur. Review each finding in context before applying fixes.
- Some `{!! !!}` usage is intentional (admin HTML content, SVGs). Use judgment.
- API routes (`routes/api.php`) legitimately skip CSRF -- do not flag them.
