---
name: lc:docker-check
description: Verify docker-compose.yml, Dockerfile, and .env are properly synchronized.
argument-hint: "[analyze | fix | fix --dry-run]"
user-invocable: true
allowed-tools: Read Grep Bash Edit Write Glob
---

# Docker Configuration Check

Verify that `docker-compose.yml`, `Dockerfile`, `.env`, and `composer.json` are properly synchronized. Detects mismatched service names, port conflicts, missing extensions, and configuration drift.

## Subcommands

| Subcommand | Description |
|---|---|
| *(no argument)* | Analyze Docker configuration and report issues. Read-only. |
| `fix` | Fix detected mismatches by syncing `.env` with docker-compose values. Asks for confirmation before each change. |
| `fix --dry-run` | Show what `.env` changes would be made without modifying anything. |

## Process

### Step 1: Read Configuration Files

Use `Read` to load:

1. `docker-compose.yml` (or `docker-compose.yaml`, `compose.yml`, `compose.yaml`)
2. `Dockerfile` (and any variant like `docker/*/Dockerfile`, `docker/Dockerfile`)
3. `.env`
4. `.env.example`
5. `composer.json` (for PHP version and extension requirements)

If any file is missing, note it and continue with available files.

### Step 2: Check Service-to-Env Synchronization

#### Database Configuration
1. Find the database service in docker-compose (mysql, mariadb, postgres).
2. Extract:
   - Service name (used as hostname): e.g., `mysql`, `db`, `database`
   - Port mapping: e.g., `3306:3306`
   - Environment variables: `MYSQL_DATABASE`, `MYSQL_USER`, `MYSQL_PASSWORD`, `MYSQL_ROOT_PASSWORD`
3. Compare with `.env`:
   - `DB_HOST` should match the service name (e.g., `mysql`, not `127.0.0.1`)
   - `DB_PORT` should match the internal port (usually `3306`)
   - `DB_DATABASE` should match `MYSQL_DATABASE`
   - `DB_USERNAME` should match `MYSQL_USER`
   - `DB_PASSWORD` should match `MYSQL_PASSWORD`

**Common issue:** `DB_HOST=127.0.0.1` when it should be the Docker service name (e.g., `mysql`).

#### Redis Configuration
1. Find the Redis service in docker-compose.
2. Compare with `.env`:
   - `REDIS_HOST` should match the service name (e.g., `redis`)
   - `REDIS_PORT` should match the internal port (usually `6379`)

#### Mail Configuration
1. Find mail services (mailhog, mailpit, mailtrap).
2. Compare with `.env`:
   - `MAIL_HOST` should match the service name
   - `MAIL_PORT` should match the internal port

#### Queue/Cache
1. Check `CACHE_DRIVER` and `QUEUE_CONNECTION` in `.env`.
2. If set to `redis`, verify Redis service exists in docker-compose.
3. If set to `database`, no extra service needed.

### Step 3: Check Port Conflicts

1. Extract all port mappings from docker-compose: `host_port:container_port`.
2. Check for duplicate host ports (would cause bind errors).
3. Check for common conflicts:
   - `3306` (MySQL) -- conflicts with local MySQL
   - `5432` (PostgreSQL) -- conflicts with local PostgreSQL
   - `6379` (Redis) -- conflicts with local Redis
   - `80`/`443` (HTTP/S) -- conflicts with local web server
   - `8080` (common dev port) -- conflicts with other dev servers

### Step 4: Check Volume Paths

1. Extract volume mounts from docker-compose.
2. For bind mounts (host path : container path):
   - Check that the host path exists or is a relative path from the project root.
   - Check that the container path makes sense for the service type.
3. For named volumes, ensure they are defined in the `volumes:` section.

### Step 5: Check PHP Version Consistency

1. Extract PHP version from Dockerfile (`FROM php:X.Y-fpm`, `FROM php:X.Y-cli`, etc.).
2. Check `composer.json` for the `require.php` constraint (e.g., `"^8.2"`).
3. Compare: the Dockerfile PHP version should satisfy the Composer constraint.
4. If there is a mismatch, flag it.

### Step 6: Check Required PHP Extensions

1. Parse `composer.json` for `require` entries starting with `ext-`:
   - `"ext-gd": "*"` -> requires `gd` extension
   - `"ext-redis": "*"` -> requires `redis` extension
   - `"ext-intl": "*"` -> requires `intl` extension
2. Parse the Dockerfile for installed extensions:
   - `docker-php-ext-install gd intl pdo_mysql`
   - `pecl install redis`
   - `RUN apt-get install` (for underlying libraries)
3. Flag extensions required by Composer but not installed in Dockerfile.

### Step 7: Check .env.example Completeness

1. Compare all keys in `.env` with keys in `.env.example`.
2. Flag keys present in `.env` but missing from `.env.example` (may cause setup issues for new developers).
3. Flag keys in `.env.example` but missing from `.env` (may indicate unused configuration).

### Step 8: Report

```
DOCKER CONFIGURATION CHECK
============================

ISSUES FOUND
--------------

1. DB_HOST mismatch
   .env: DB_HOST=127.0.0.1
   docker-compose: Database service name is 'mysql'
   Fix: Change DB_HOST to 'mysql' in .env
   Severity: Critical (app cannot connect to DB in Docker)

2. Port conflict
   Service 'mysql' maps to host port 3306
   Service 'testing-db' also maps to host port 3306
   Fix: Change one service to a different host port (e.g., 3307:3306)
   Severity: Critical (docker-compose will fail to start)

3. Missing PHP extension
   composer.json requires: ext-redis
   Dockerfile: redis extension not found
   Fix: Add 'pecl install redis && docker-php-ext-enable redis' to Dockerfile
   Severity: High (application will fail at runtime)

4. PHP version mismatch
   Dockerfile: PHP {detected_version}
   composer.json: "php": "{composer_constraint}"
   Status: Compatible/Incompatible

5. Missing .env.example keys
   Keys in .env but not in .env.example:
   - GOOGLE_OAUTH_CLIENT_ID
   - GOOGLE_OAUTH_CLIENT_SECRET
   - MICROSOFT_OAUTH_CLIENT_ID
   Severity: Low (documentation issue)

ALL CHECKS
-----------
[PASS] Database service exists and is properly configured
[FAIL] DB_HOST does not match service name
[PASS] Redis service exists and matches REDIS_HOST
[PASS] No port conflicts detected
[PASS] Volume paths are valid
[PASS] PHP version is compatible
[FAIL] Missing ext-redis in Dockerfile
[WARN] 3 .env keys missing from .env.example

SUMMARY
========
Checks performed: 8
Passed: 5
Failed: 2
Warnings: 1
```

## Fix Mode

In fix mode, apply corrections with confirmation:

1. **Sync .env values**: Update `DB_HOST`, `REDIS_HOST`, `MAIL_HOST` to match docker-compose service names.
2. **Sync credentials**: Update `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` to match docker-compose environment variables.
3. **Update .env.example**: Add missing keys with placeholder values.

**Not auto-fixed (manual intervention needed):**
- Dockerfile changes (adding extensions, changing PHP version)
- Port conflict resolution (requires understanding of the developer's environment)
- Volume path fixes

## Notes

- This skill reads configuration files only. It does not start, stop, or modify Docker containers.
- Docker Compose file format (version 2 vs 3) affects syntax. Handle both formats.
- Some projects use multiple Dockerfiles (e.g., `docker/{version}/Dockerfile` for different PHP versions). Check all.
- `.env` files may have platform-specific line endings. Handle both `\n` and `\r\n`.
- When fixing `.env`, preserve the existing file format and ordering.
