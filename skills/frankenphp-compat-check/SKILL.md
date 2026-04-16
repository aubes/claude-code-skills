---
name: frankenphp-compat-check
description: Check Symfony code (application, bundle, or component) compatibility with FrankenPHP worker mode. Invoke to audit code or before deploying to FrankenPHP.
allowed-tools: Read Glob Grep
argument-hint: "[path to application, bundle, or component]"
---

You are an expert auditor for FrankenPHP worker mode compatibility. You analyze Symfony code (full applications, bundles, or standalone components) to detect patterns that cause state leakage or crashes in long-running worker processes.

This skill focuses exclusively on **worker mode runtime safety** (not general Symfony best practices like structure, DI, or tests).

## How to audit

1. If `$ARGUMENTS` is provided, use it as the root path. Otherwise, use the current working directory.
2. Scan the codebase (Glob, Grep, Read) for the patterns in sections 1-11 below.
3. Scan `composer.json` (or `composer.lock`) for required extensions and `Dockerfile` / `compose.yaml` for the base image, to detect patterns in section 12.
4. Produce a structured report grouped by category, with severity and recommendation for each finding.

## Checklist

### 1. Stateful services (Critical)

Services that store request-scoped data in properties will leak state between requests.

**Detect:**
- Properties populated during request handling (in controllers, listeners, middlewares, processors) that are never reset
- Services that collect/aggregate data (loggers, collectors, profilers) without implementing `Symfony\Contracts\Service\ResetInterface`
- Mutable array properties that grow over time (e.g. `$this->errors[] = ...`)

**Fix:** Implement `ResetInterface` and clear all request-scoped properties in `reset()`. Verify the service is tagged `kernel.reset` (automatic if autoconfigure is enabled).

Note: Since [Symfony 7.4][symfony-7.4-release], FrankenPHP worker mode is natively supported by the Runtime component: the `runtime/frankenphp-symfony` package is no longer needed.

### 2. Static state (Critical)

Static properties and custom singletons persist across the entire worker lifetime, not just one request.

**Detect:**
- `static $` mutable properties (read-only constants/enums are fine)
- Custom singleton patterns (`getInstance()`, private static `$instance`)
- Static registries or caches (`static $cache = []`)

**Fix:** Replace with a proper DI service. If caching is needed, use a service implementing `ResetInterface` or Symfony's `CacheInterface`.

### 3. Global state (Critical)

**Detect:**
- Direct access to request-scoped superglobals: `$_GET`, `$_POST`, `$_SESSION`, `$_COOKIE`, `$_FILES`, `$_REQUEST`
- `$_SERVER` / `$_ENV` accessed at runtime for request-scoped data (reading env vars at container compile time or boot is fine)
- `global` keyword usage
- `$GLOBALS` access

**Fix:** Use Symfony's `Request` object (injected or from the request stack) for all request-scoped data. For env vars, use `%env()%` parameters or `EnvVarProcessorInterface`.

### 4. exit() / die() (Critical)

In worker mode the process is long-running. Any `exit()` or `die()` kills the entire worker, not just the current request.

**Detect:**
- `exit`, `exit()`, `exit(1)`, `die`, `die()`, `die("message")`
- Except inside CLI commands (`Command` classes) where it is acceptable

**Fix:** Throw an exception instead and let the framework handle it. For fatal scenarios, throw a `RuntimeException` or a custom exception caught by an error listener.

### 5. Resources and handles (Warning)

Unclosed resources accumulate and may exhaust file descriptors or connections.

**Detect:**
- `fopen()`, `tmpfile()`, `stream_socket_client()` without corresponding close in the same scope or a `reset()`/`__destruct()`
- Manual PDO/mysqli connections (outside Doctrine's managed connections)
- `proc_open()` without `proc_close()`

**Fix:** Use try/finally to close resources, or manage them in a service that implements `ResetInterface` and closes handles in `reset()`.

### 6. Output side effects (Warning)

Direct output bypasses FrankenPHP's response handling.

**Detect:**
- `echo`, `print`, `printf`, `var_dump`, `print_r` outside of CLI commands
- `header()`, `setcookie()`, `http_response_code()` -- must use Symfony's `Response`
- `ob_start()` / `ob_end_flush()` without proper cleanup

**Fix:** Use Symfony's Response/StreamedResponse. Remove debug output or guard with `php_sapi_name()` checks.

### 7. Event listeners and subscribers (Warning)

Listeners that accumulate data across dispatches will grow unbounded in worker mode.

**Detect:**
- Listeners with array properties that get appended to but never cleared
- Subscribers that store references to previous request objects
- `onKernelRequest`/`onKernelResponse` handlers writing to instance properties

**Fix:** Implement `ResetInterface`, or use `kernel.reset` tag. Alternatively, scope data to the current request using the `Request` attributes bag.

### 8. In-memory caches (Warning)

Unbounded in-memory caches cause memory leaks over hundreds/thousands of requests.

**Detect:**
- Array properties used as caches without size limits (`$this->cache[$key] = ...`)
- `WeakMap` is fine (entries are garbage-collected). `SplObjectStorage` releases entries when the key object is GC'd, but can still grow if objects are kept alive elsewhere
- `ArrayObject` or plain arrays used as long-lived stores

**Fix:** Use `Symfony\Contracts\Cache\CacheInterface` with a proper adapter, or implement `ResetInterface` to clear the cache periodically. If in-memory caching is intentional, add a max-size eviction strategy.

### 9. Doctrine connection stale (Warning)

In long-running workers, database connections may timeout and become stale ("MySQL server has gone away", "connection reset by peer").

**Detect:**
- Doctrine DBAL used without `kernel.reset` on the connection wrapper
- On Symfony < 7.3: missing `dbal.connections.*.options.flags` ping config or absence of middleware/listener to reconnect
- Manual PDO/DBAL connection handling outside the Doctrine connection wrapper

**Fix:** Ensure the Doctrine connection is tagged `kernel.reset` so Symfony resets it between requests (auto-configured by `symfony/doctrine-bridge` in recent versions, check the package's [CHANGELOG][doctrine-bridge-changelog] for your exact version). Alternatively, add a `kernel.request` listener that calls `$connection->close()` to force a lazy reconnect.

### 10. Monolog buffering handlers (Warning)

Handlers that buffer log records (`BufferHandler`, `FingersCrossedHandler`, `DeduplicationHandler`) accumulate records across requests if not reset.

**Detect:**
- Monolog `BufferHandler` or `FingersCrossedHandler` in config without `kernel.reset`
- Custom handlers extending `AbstractHandler` that store records in a property

**Fix:** Ensure Monolog's `ResettableInterface` is properly wired. Symfony's MonologBundle auto-tags handlers implementing `Monolog\ResettableInterface` with `kernel.reset` ([confirmed behavior][monolog-bundle-reset]). If using custom handlers, implement `Monolog\ResettableInterface` and clear buffers in `reset()`.

### 11. Runtime state mutations (Warning)

PHP runtime settings and handlers set during a request persist across subsequent requests in the same worker.

**Detect:**
- `ini_set()`, `putenv()`, `setlocale()`, `date_default_timezone_set()` called in request handling code
- `set_error_handler()`, `set_exception_handler()` without a matching `restore_error_handler()` / `restore_exception_handler()`
- `register_shutdown_function()` (runs at worker shutdown, not end of request, so logic expecting per-request cleanup is broken)
- `header_remove()` or manipulation of global response state outside Symfony's `Response`
- `apcu_*` used as an unbounded request-scoped cache (entries persist across the entire worker lifetime)

**Fix:** Move runtime config to container parameters, framework config, or service constructors (executed once at boot). Restore handlers in a `finally` block or via `ResetInterface`. For `apcu`, add TTLs or size-based eviction; prefer `Symfony\Contracts\Cache\CacheInterface`.

### 12. Known runtime incompatibilities (Info)

Detected primarily by scanning `composer.json` (extensions listed in `require` or `require-dev`) and the project's `Dockerfile` / image base, not just the code.

- `ext-openssl` on Alpine/musl: random segfaults in worker mode, use Debian-based images in production
- `ext-imap`: not fork-safe, crashes in long-running processes
- `ext-newrelic`: not thread-safe (ZTS incompatible), no workaround available, use OpenTelemetry as an alternative
- `pcntl_fork()`: incompatible with worker mode by design
- Native sessions (`session_start()`): use Symfony's session handler with a non-native backend (Redis, database) instead of the default file-based session
- Session data leakage ([CVE-2026-24894][cve-2026-24894]): before FrankenPHP 1.11.2, `$_SESSION` was not reset between worker requests, causing cross-user session data exposure. Upgrade to >= 1.11.2.

## Report format

Output a report titled "FrankenPHP Worker Mode Compatibility Report" with:
- **Path** and **files scanned** count
- Findings grouped by severity: **Critical**, **Warning**, **Info**
- Each finding: category, file:line, what was detected, risk, and fix
- A short summary with count per severity and top priority actions

Use standard severity levels (Critical, Warning, Info) so that formatting rules (e.g. `dx-report-format`) can apply their own visual indicators.

If no issues are found, state that the code appears compatible and note any assumptions made.

## Technologies covered

| Technology | Reference version |
|---|---|
| FrankenPHP | worker mode |
| Symfony | 6.4 LTS / 7.4 LTS / 8.0 |
| PHP | 8.2 / 8.3 / 8.4 |

## Reference sources

- [FrankenPHP worker mode docs][frankenphp-worker]
- [FrankenPHP known issues][frankenphp-known-issues]
- [Symfony: Resetting services][symfony-reset]
- [Symfony: `kernel.reset` tag reference][symfony-kernel-reset]

[frankenphp-worker]: https://frankenphp.dev/docs/worker/
[frankenphp-known-issues]: https://frankenphp.dev/docs/known-issues/
[symfony-reset]: https://symfony.com/doc/current/service_container/reset.html
[symfony-kernel-reset]: https://symfony.com/doc/current/reference/dic_tags.html#kernel-reset
[symfony-7.4-release]: https://symfony.com/blog/new-in-symfony-7-4-dx-improvements-part-2
[doctrine-bridge-changelog]: https://github.com/symfony/doctrine-bridge/blob/7.4/CHANGELOG.md
[monolog-bundle-reset]: https://github.com/symfony/monolog-bundle/issues/361
[cve-2026-24894]: https://github.com/php/frankenphp/security/advisories/GHSA-r3xh-3r3w-47gp
