---
name: lc:deploy-checklist
description: Verify the project is production-ready with a comprehensive checklist.
argument-hint: "[analyze]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Deploy Checklist

Verify that a Laravel project is production-ready by checking configuration, security, performance, and code quality. Reports a pass/fail checklist with remediation steps for each failure.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Run the full deployment readiness checklist. Read-only report. |

This is an analysis-only skill. It reports findings but does not modify files. Use the suggested remediation steps or other skills (e.g., `lc:security-audit`, `lc:docker-check`) to fix issues.

## Checklist Categories

### 1. Environment Configuration

#### APP_DEBUG
- **Check**: `Read` `.env` and verify `APP_DEBUG=false`.
- **Also check**: `config/app.php` uses `env('APP_DEBUG', false)` (not hardcoded `true`).
- **PASS**: `APP_DEBUG=false` or not set (defaults to false)
- **FAIL**: `APP_DEBUG=true`
- **Why it matters**: Debug mode exposes sensitive information (stack traces, env variables, database credentials) to end users.

#### APP_ENV
- **Check**: `Read` `.env` and verify `APP_ENV=production`.
- **PASS**: `APP_ENV=production`
- **FAIL**: `APP_ENV=local` or `APP_ENV=staging`
- **Why it matters**: Some packages and features behave differently in production vs other environments.

#### APP_KEY
- **Check**: `APP_KEY` is set and not empty in `.env`.
- **PASS**: `APP_KEY=base64:...` (non-empty, properly formatted)
- **FAIL**: `APP_KEY=` (empty) or missing
- **Why it matters**: The app key is used for encryption. Without it, sessions, encrypted cookies, and other encrypted data are insecure.

#### APP_URL
- **Check**: `APP_URL` uses `https://` (not `http://`).
- **PASS**: Starts with `https://`
- **FAIL**: Starts with `http://` or is `http://localhost`
- **Why it matters**: Insecure URLs affect generated links, asset URLs, and OAuth callbacks.

### 2. Cache Configuration

#### Cache Driver
- **Check**: `CACHE_STORE` (or `CACHE_DRIVER`) in `.env`.
- **PASS**: `redis`, `memcached`, `database`, or `dynamodb`
- **WARN**: `file` (works but slower in production, not suitable for multi-server)
- **FAIL**: `array` or `null` (no persistent caching)
- **Why it matters**: File cache is slow and doesn't work across multiple servers.

#### Session Driver
- **Check**: `SESSION_DRIVER` in `.env`.
- **PASS**: `redis`, `memcached`, `database`
- **WARN**: `file` (works for single-server deployments)
- **FAIL**: `array` or `cookie` (no persistence or security concerns)

### 3. Queue Configuration

#### Queue Driver
- **Check**: `QUEUE_CONNECTION` in `.env`.
- **PASS**: `redis`, `database`, `sqs`, `beanstalkd`
- **FAIL**: `sync` (jobs execute inline, blocking the request)
- **Why it matters**: Sync queue means email sending, file processing, and other jobs block HTTP requests.

### 4. Mail Configuration

#### Mail Driver
- **Check**: `MAIL_MAILER` in `.env`.
- **PASS**: `smtp`, `ses`, `mailgun`, `postmark`, `sendgrid`
- **FAIL**: `log` (emails only written to log file, not sent)
- **FAIL**: `array` (emails discarded)
- **Why it matters**: Users will not receive emails (password resets, notifications, invitations).

### 5. Security Checks

#### HTTPS Enforcement
- **Check**: Look for HTTPS enforcement mechanisms:
  - `URL::forceScheme('https')` in `AppServiceProvider`
  - `HTTPS` middleware or `\Illuminate\Http\Middleware\TrustProxies`
  - `.htaccess` or nginx config redirecting HTTP to HTTPS
- **PASS**: At least one HTTPS enforcement mechanism found
- **FAIL**: No HTTPS enforcement detected

#### CORS Configuration
- **Check**: `Read` `config/cors.php`.
- **PASS**: `allowed_origins` is specific (not `['*']` in production)
- **WARN**: `allowed_origins` is `['*']` (allows all origins)
- **Why it matters**: Wildcard CORS allows any website to make requests to your API.

#### Rate Limiting on Auth Routes
- **Check**: Look for rate limiting on login/registration routes.
- Use `Grep` to find `throttle` middleware on auth routes.
- **PASS**: Auth routes have `throttle` middleware
- **FAIL**: No rate limiting on auth routes
- **Why it matters**: Without rate limiting, brute-force attacks are possible.

### 6. Storage and Files

#### Storage Link
- **Check**: Verify `public/storage` symlink exists.
- Use `Bash`: `ls -la public/storage`
- **PASS**: Symlink exists and points to `storage/app/public`
- **FAIL**: Symlink missing (run `php artisan storage:link`)

### 7. Code Quality

#### TODO/FIXME/HACK Comments
- **Check**: Use `Grep` to find `TODO`, `FIXME`, `HACK`, `XXX`, `TEMP` in PHP and Blade files.
- **PASS**: Zero found
- **WARN**: Found (list them with file paths and line numbers)
- **Why it matters**: These indicate incomplete or temporary code that should be resolved before production.

#### Debug Statements
- **Check**: Use `Grep` to find debug statements that should not be in production:
  - `dd(`, `dump(`, `ray(` in PHP files
  - `console.log(` in JS files (except in build/vendor)
  - `var_dump(`, `print_r(` in PHP files
- **PASS**: Zero found
- **FAIL**: Debug statements found (list them)

#### .env in Version Control
- **Check**: Verify `.env` is in `.gitignore`.
- Use `Grep` to check `.gitignore` for `.env` entry.
- **PASS**: `.env` is gitignored
- **FAIL**: `.env` is not gitignored (security risk)

### 8. Database

#### Migrations Can Run
- **Check**: If Docker is available, note whether migrations appear consistent (do not actually run migrate:fresh -- that is destructive).
- Look for common migration issues by parsing files (FK ordering, duplicate columns).
- **PASS**: No obvious migration issues detected
- **WARN**: Potential issues found (reference `lc:migration-fresh-test` for full test)

#### Schema Dump
- **Check**: Look for `database/schema/*.sql` files.
- **WARN**: Schema dump exists (may cause issues with migrate:fresh, data imports may be lost)
- **PASS**: No schema dump

### 9. Performance

#### Config Caching
- **Check**: Note whether config, route, and view caching commands should be run in deployment.
- Verify `config/app.php` does not use `env()` outside of config files (breaks config caching).
- Use `Grep` to find `env(` calls in `app/`, `routes/`, `resources/` directories.
- **PASS**: No `env()` calls outside config files
- **FAIL**: `env()` used outside config (breaks config caching)

#### Eager Loading
- **Note**: Reference `lc:find-n-plus-one` for N+1 query detection.

## Report Format

```
DEPLOYMENT READINESS CHECKLIST
================================

ENVIRONMENT
  [PASS] APP_DEBUG = false
  [FAIL] APP_ENV = local (should be 'production')
  [PASS] APP_KEY is set
  [FAIL] APP_URL uses http:// (should be https://)

CACHE & SESSIONS
  [PASS] CACHE_STORE = redis
  [PASS] SESSION_DRIVER = redis

QUEUE
  [FAIL] QUEUE_CONNECTION = sync (should be redis/database/sqs)

MAIL
  [PASS] MAIL_MAILER = ses

SECURITY
  [PASS] HTTPS enforcement found (URL::forceScheme)
  [WARN] CORS allows all origins (config/cors.php)
  [PASS] Rate limiting on auth routes
  [PASS] .env is gitignored

STORAGE
  [PASS] Storage symlink exists

CODE QUALITY
  [WARN] 3 TODO comments found
  [FAIL] 2 dd() statements found
    - app/Http/Controllers/DashboardController.php:45
    - app/Services/PropertyService.php:102

DATABASE
  [PASS] No obvious migration issues
  [WARN] Schema dump exists at database/schema/mysql-schema.sql

PERFORMANCE
  [FAIL] env() used outside config files:
    - app/Services/StripeService.php:15
    - app/Http/Middleware/CustomMiddleware.php:8

SUMMARY
========
Passed:   10
Warnings:  3
Failed:    4
Score:     59% ready

PRIORITY FIXES (do these first):
1. Set APP_ENV=production
2. Set QUEUE_CONNECTION=redis
3. Remove dd() statements
4. Move env() calls to config files
```

## Notes

- This checklist is based on static analysis and configuration file inspection. Some checks require runtime verification.
- Not all warnings are blockers. Some are acceptable trade-offs for specific deployments (e.g., `file` cache on a single-server setup).
- The checklist does not check infrastructure concerns (SSL certificates, server configuration, DNS, CDN).
- For security-specific deep analysis, use `lc:security-audit`.
- For Docker configuration, use `lc:docker-check`.
- For migration testing, use `lc:migration-fresh-test`.
