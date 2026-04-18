---
name: symfony-frankenphp-check
description: Audit a Symfony application, bundle, or component for FrankenPHP worker mode compatibility. Invoke to review code or before deploying to FrankenPHP.
allowed-tools: Read Glob Grep
argument-hint: "[path to application, bundle, or component] [--critical-only]"
---

You are an expert auditor for FrankenPHP worker mode compatibility. You analyze Symfony code (full applications, bundles, or standalone components) to detect patterns that cause state leakage or crashes in long-running worker processes.

This skill focuses exclusively on **worker mode runtime safety** (not general Symfony best practices like structure, DI, or tests).

**Adjacent scopes (not covered here, but most rules transfer):** Symfony Messenger consumers (`messenger:consume`) and Symfony Scheduler (`#[AsCronTask]`, `#[AsPeriodicTask]`) are also long-running PHP processes. Sections §1, §2, §4, §5, §8, §9, §10, §11 apply almost verbatim (substitute "request" with "message" and `kernel.reset` with the Messenger `ServicesResetter` wired on `WorkerRunningEvent`). Sections §3 (superglobals), §6 (HTTP output), §7 (kernel event listeners) and §12 (FrankenPHP-specific extensions/CVEs) are HTTP-worker-specific and do not transfer. OS-level cronjobs running `bin/console` one-shot are out of scope (fresh process per run, no leak possible).

## Note on `ResetInterface` fixes

Several sections below (§1, §5, §7, §8, §9, §10) recommend implementing `ResetInterface` or tagging `kernel.reset` as the fix for per-request state. This only works if the service is actually instantiated during the request: a dedicated resetter nobody injects is silently skipped by Symfony's `ServicesResetter`. See §1 "Note on `reset()` placement" before recommending the implementation on a standalone class.

## How to audit

1. If `$ARGUMENTS` is provided, use it as the root path. Otherwise, use the current working directory. Detect the `--critical-only` flag (or an explicit natural request like "critical only", "quick audit") to restrict the audit to Critical bullets; otherwise audit all severities.
2. **Confirm scan scope before any scan**:
   - List the top-level directories under the root path. Show which will be scanned and which are excluded by default (`vendor/`, `var/cache/`, `var/log/`, `tests/`, `node_modules/`).
   - Show the audit mode (full audit, or critical-only).
   - **STOP here and wait for an explicit response** on both scope and mode. Do not call any Grep or Read tool until the developer has replied.
3. Scan the codebase (Glob, Grep, Read) for the patterns in sections 1-11 below:
   - **Full audit** (default): scan all bullets of sections 1-11.
   - **Critical-only mode**: scan only bullets labelled Critical across all sections (including the Critical sub-blocks of §3 and §12).
   - In both modes, cross-section routing rules (e.g. "report here only if it doesn't fit §7 or §8") stay active: classify each finding under its most specific section, but emit only findings whose severity matches the selected mode.
4. Scan `composer.json` (or `composer.lock`), `Dockerfile` / `compose.yaml`, and `php.ini` to detect patterns in section 12. In critical-only mode, restrict to §12 Critical bullets.
5. Produce a structured report grouped by category, with severity and recommendation for each finding.

## Runtime setup

Before auditing code patterns, verify the project's runtime wiring (read `composer.json` / `composer.lock`):

- **PHP version**: FrankenPHP requires PHP >= 8.2. Flag any `require.php` constraint that allows versions below 8.2 (e.g. `"php": "^8.1"`).
- **Symfony ≥ 7.4**: FrankenPHP worker mode is natively supported by the Runtime component ([release notes][symfony-7.4-release]). The `runtime/frankenphp-symfony` package is no longer needed and can be removed from `composer.json`.
- **Symfony 6.4 / 7.0-7.3**: the `runtime/frankenphp-symfony` package is required to bridge the Runtime component with FrankenPHP's worker loop.

Flag any mismatch (e.g. Symfony 7.4 still depending on `runtime/frankenphp-symfony`, or Symfony 7.2 without it) as an **Info** finding. Flag a PHP constraint below 8.2 as a **Warning**. The FrankenPHP binary version itself is checked in §12 (CVE-2026-24894).

## Checklist

### 1. Stateful services (Critical)

**Scope:** this section covers the general case of services mutating their own state during request handling. Two sub-cases have dedicated sections with more specific detection patterns:
- Event listeners and subscribers → see §7
- In-memory caches (keyed stores, memoization) → see §8

**Key distinction:**
- ✅ Properties set once at construction or container boot (config, injected dependencies, precomputed data) are safe: they never change between requests.
- ❌ Properties mutated while handling a request (assigned, appended, incremented) are unsafe unless the service is reset between requests.

**Detect:**
- Property assignments or mutations (`$this->foo = ...`, `$this->items[] = ...`, `$this->counter++`) in any service called per request (controllers, listeners, message handlers, voters, state providers/processors, etc.)
- Services aggregating data across calls (collectors, profilers, custom loggers) that do not implement `Symfony\Contracts\Service\ResetInterface`
- Properties typed as mutable collections (`array`, `ArrayObject`, `SplQueue`) that grow during request handling
- Services tagged `kernel.reset` or implementing `ResetInterface` with no DI consumer (no constructor injection, no `service()` / `@` reference, no `ServiceLocator` entry anywhere in the codebase): their `reset()` never fires and per-request state leaks across requests. See "Note on `reset()` placement" below for the mechanism and fix.

**Detect procedure (orphan resetters):**
1. Grep for classes implementing `ResetInterface` or tagged `kernel.reset` (including `#[AsTaggedItem('kernel.reset')]` / `#[AutoconfigureTag('kernel.reset')]`).
2. For each, grep its FQCN across all service definitions (`*.yaml`, `*.php`, `*.xml`) and across constructors and parameters (type-hints, autowiring, `#[Autowire]`, `ServiceLocator` entries).
3. If no consumer is found, flag it as an orphan resetter.

**Ignore:**
- Assignments in `__construct()`, `setContainer()`, or boot-time hooks
- `readonly` properties and properties only written once then treated as immutable
- Request-scoped services (explicitly tagged or scoped per request)

**Fix:** Implement `ResetInterface` and clear all request-scoped properties in `reset()`. The `kernel.reset` tag is applied automatically when autoconfigure is enabled; verify it with `debug:container --tag=kernel.reset`.

**Note on `reset()` placement:** `kernel.reset` fires only on services actually instantiated during the request. A dedicated resetter class nobody injects stays uninitialized and its `reset()` is silently skipped (`ServicesResetter` uses `IGNORE_ON_UNINITIALIZED_REFERENCE`). Idiomatic placements:
- Implement `ResetInterface` directly on the service that mutates the state, or on a listener guaranteed to be instantiated each request (e.g. a `kernel.request` listener).
- If the resetter must stay a dedicated class, inject it into a service that is consumed on every request, so the container materializes it.
- Verify with `debug:container --tag=kernel.reset`, then confirm end-to-end that `reset()` fires (e.g. a temporary `error_log()` inside `reset()`); tag presence alone does not guarantee the call.

### 2. Static state (Critical)

**Scope:** state held in static properties or class-level singletons. Related sections:
- Static properties used as keyed caches (lookup, memoization) → §8
- Instance-level stateful services → §1

**Detect:**
- Static property declarations with mutable state: `private static array $items`, `public static ?self $instance`
- Writes to static properties during request handling: `self::$foo = ...`, `static::$cache[$key] = ...`, `ClassName::$registry[] = ...`
- Static methods that mutate class state: `Registry::register(...)`, `Logger::addHandler(...)` when the target is a static property
- Custom singleton patterns: `getInstance()`, `getShared()`, private static `$instance` combined with a private constructor
- Late static binding used for shared state: `static::` on mutable class properties

**Ignore:**
- `const` and `enum` cases: immutable by design
- Static properties assigned once at class load and never reassigned (e.g. lookup tables built from a literal)
- Memoization with a bounded key domain: `static $cache = []` inside a pure function keyed on a closed set of values (enum, class name, small int range). Flag if the key domain is user input or otherwise unbounded.
- Singletons from third-party vendor code you cannot modify. Report these under §12 (known runtime incompatibilities) with a note on how the caller isolates them.

**Fix:** Replace with a proper DI service. If caching is needed, use a service implementing `ResetInterface` or Symfony's `CacheInterface`.

### 3. Global state (Critical / Warning)

**Scope:** direct reads and writes to PHP superglobals and the `global` keyword. Related sections:
- Runtime setting mutations (`putenv()`, `ini_set()`, `setlocale()`...) → §11
- `$_SESSION` leakage on FrankenPHP < 1.11.2 → §12 (CVE-2026-24894)

In FrankenPHP worker mode, most superglobals (`$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_REQUEST`, `$_SERVER`) are repopulated by the runtime on each request. *Reading* them is not a state leak but still bypasses Symfony's Request abstraction (testability, coupling). *Writing* to globals persists across requests and is a real leak.

**Detect — Critical (writes persist across requests):**
- Writes to `$GLOBALS`: `$GLOBALS['foo'] = ...`
- `global $x; $x = ...` pattern that mutates the global
- Writes to `$_ENV` or `$_SERVER` at runtime

**Detect — Warning (bad practice, not a leak in worker mode):**
- Reads or writes to `$_SESSION`: bypasses Symfony's `SessionInterface`. **Escalate to Critical** if the target FrankenPHP version is < 1.11.2 (see §12, CVE-2026-24894).
- Reads from request superglobals: `$_GET`, `$_POST`, `$_COOKIE`, `$_FILES`, `$_REQUEST`
- Reads from `$_SERVER` for request-scoped data (headers, method, URI)
- Reads from `$_ENV` for request-scoped data (env vars at boot time are fine)
- `global $x;` read-only usage

**Fix:**
- **Request data**: inject `Request` as a controller argument or get it from `RequestStack`. Use `$request->query`, `$request->request`, `$request->cookies`, `$request->files`, `$request->headers`, `$request->server` instead of the matching superglobals.
- **Typed mappers (Symfony 6.3+)**: `#[MapQueryString]`, `#[MapRequestPayload]`, `#[MapQueryParameter]` on controller arguments for automatic DTO mapping with validation.
- **Sessions**: never touch `$_SESSION`. Use `$request->getSession()` which returns a `SessionInterface`.
- **Env vars**: `%env()%` parameters for config, `EnvVarProcessorInterface` for custom processors, or inject config-scoped values via DI. Read env vars once at boot, never during request handling.
- **Shared state** (the `global` / `$GLOBALS` case): turn it into a proper service registered in the container; apply §1 or §2 recommendations depending on whether the state is instance- or class-scoped.

### 4. exit() / die() (Critical)

**Scope:** abrupt process termination calls in first-party code. Vendor code calling `exit()` is out of scope here; escalate to §12 as a known runtime incompatibility.

In worker mode the process is long-running. Any `exit()` or `die()` kills the entire worker, not just the current request.

**Detect:**
- `exit` or `die` calls (with or without arguments)
- `pcntl_exec()`: replaces the worker process entirely
- `posix_kill(posix_getpid(), ...)`: sends a signal to the worker itself
- `trigger_error($msg, E_USER_ERROR)`: fatal error halts execution unless a global error handler catches it

**Ignore:**
- Symfony `Command` classes: execute() returns an exit code, and `exit()` there is acceptable
- Entry-point scripts: `public/index.php`, `bin/console`, custom `bin/*` scripts
- Vendor code (`vendor/`): outside the audit scope; if a vendor library calls `exit()` in a hot path, escalate to §12 as a known runtime incompatibility

**Fix:** replace with the idiomatic abort for the current context.
- **Controller**: return a `Response` with the right status code, or throw `HttpException` / `NotFoundHttpException` / `AccessDeniedHttpException`.
- **Service or business logic**: throw a domain exception. The framework's exception listener converts it to a response.
- **`kernel.request` / `kernel.response` listener**: call `$event->setResponse(...)` to short-circuit without killing the worker.
- **`kernel.exception` listener**: call `$event->setResponse(...)` on the event. Do not throw, you are already inside error handling.
- **Messenger handler**: throw to nack/retry; Messenger handles the rest.
- **True fatal** (e.g. corrupted worker state): let the exception bubble up and rely on the supervisor (PM2, systemd, FrankenPHP's own watchdog) to restart the worker cleanly. Never call `exit()` yourself.

### 5. Resources and handles (Warning)

**Scope:** file handles, sockets, network clients, and manually managed DB connections. Related sections:
- Doctrine-managed DB connection lifecycle (staleness, ping/reconnect) → §9
- In-memory caches that happen to wrap a resource (e.g. a Redis client used as a cache) → §8

Unclosed resources accumulate and may exhaust file descriptors or connections.

**Detect:**
- File and stream handles: `fopen()`, `tmpfile()`, `fsockopen()`, `stream_socket_client()`, `stream_socket_server()` without a close in the same scope or in `ResetInterface::reset()`
- Process handles: `proc_open()`, `popen()` without their matching close
- Network handles: `curl_init()`, `socket_create()`, `ftp_connect()`, `ssh2_connect()` held on long-lived service properties without close. PHP 8+ auto-closes these on GC, but GC may never trigger on a property that outlives the worker.
- Manual DB connections outside Doctrine: PDO, mysqli, `pg_connect()`
- Persistent connections (`pg_pconnect`, `mysqli_pconnect`, PDO `ATTR_PERSISTENT`): survive across requests by design. Document the intent and see §9 for staleness handling.
- Cache/NoSQL clients (Redis, Memcached) held on service properties without a reset strategy
- Resources released only in `__destruct()`: unreliable in worker mode (see Fix)

**Ignore:**
- `php://temp` and `php://memory` streams: auto-managed by PHP, no fd leak risk
- Symfony's high-level wrappers (`Filesystem`, `File`, `UploadedFile`): handle their own lifecycle
- Doctrine-managed connections: see §9 instead
- Dev-only fixtures folders (`fixtures/` at project root): outside worker runtime

**Fix:**
- **Short-lived resources** (opened and used within one method): wrap in try/finally and close in the finally block.
- **Long-lived resources** (held on a service property): implement `ResetInterface` and close in `reset()`.
- **Warning**: do not rely on `__destruct()` in worker mode. Container services are destroyed only when the worker shuts down, not per request, so resources held on them can leak for thousands of requests before cleanup.

### 6. Output side effects (Warning)

**Scope:** direct output and HTTP wire calls that bypass Symfony's `Response`. Related sections:
- `header_remove()` and raw HTTP response state manipulation → §11 (runtime state angle)
- Output buffers persisting across requests → detailed here (see Note below)

**Detect:**
- Direct output: `echo`, `print`, `printf`, `vprintf`, `phpinfo()`, `debug_print_backtrace()`
- `var_dump()`, `print_r($x)`, `var_export($x)`: output to stdout
- Direct stream writes: `fwrite(STDOUT, ...)`, `fwrite(STDERR, ...)`, `fputs(STDOUT, ...)`
- HTTP wire calls outside Symfony's Response: `header()`, `setcookie()`, `setrawcookie()`, `http_response_code()`, `header_register_callback()`
- Forced flushes: `flush()`, `ob_flush()` during request handling
- `ob_start()` without a guaranteed matching `ob_end_*()` (try/finally)
- `echo` / `print` inside a Twig extension (`TwigFunction`, `TwigFilter`, `TwigRuntime`) callback: idiomatically these should return a string. If the callback wraps the echo in `ob_start()` / `ob_get_clean()`, verify the try/finally guarantee (see rule above).
- `Twig\Node::compile()` implementations emitting `ob_start()` / `ob_get_clean()` via `$compiler->write(...)` **without** emitting a matching `try { ... } finally { ... }` around them: the compiled template runs in the worker loop, so an exception thrown during the captured body leaves the buffer stacked across subsequent requests. This pattern is widespread in community tutorials and in the legacy [`twig/twig-cache-extension`][twig-cache-extension-node]; the modern [`twig/cache-extra`][twig-cache-extra-node] avoids it entirely via `CaptureNode` + a `Runtime` service.

**Note on output buffers:** the worker is a single long-lived PHP process. An `ob_start()` without a matching `ob_end_clean()` / `ob_end_flush()` stays on the buffer stack and captures output from every subsequent request served by that worker. Always wrap `ob_start()` in a try/finally that closes the buffer, even on exception.

**Ignore:**
- Output inside a `StreamedResponse` callback: the framework expects `echo` / `print` to stream the body
- `print_r($x, true)` and `var_export($x, true)`: with `true` as the second argument they return a string instead of outputting (no equivalent for `var_dump()`)
- Symfony `Command` classes: output goes to the console, not the HTTP response
- `error_log()` to the default destination (PHP error log / syslog): not stdout
- `symfony/var-dumper` `dump()`: in dev, goes through the VarDumper server. Note: `dd()` also calls `die()` and is flagged by §4.
- `Twig\Node::compile()` implementations calling `$compiler->write('echo ...')`: this is code generation (compile-time), not direct runtime output. The emitted code is still subject to the other rules above: an emitted `ob_start()` without a matching emitted try/finally must be flagged (see the dedicated Detect bullet).

**Fix:**
- **Response body**: return `Response`, `JsonResponse`, `BinaryFileResponse`, or `StreamedResponse` (the only place where `echo` / `print` is expected).
- **Headers**: `$response->headers->set('X-Foo', 'bar')` instead of `header()`.
- **Cookies**: `$response->headers->setCookie(Cookie::create('name', 'value'))` instead of `setcookie()`.
- **Status code**: pass to the `Response` constructor or use `$response->setStatusCode(404)`, not `http_response_code()`.
- **Debug output**: use the `logger` service, or `symfony/var-dumper`'s `dump()` (routes through the VarDumper server in dev). To guard dev-only output, inject `%kernel.environment%` or check a feature flag. Do not rely on `php_sapi_name()`: under FrankenPHP it returns `"frankenphp"` in both dev and prod, so it cannot be used as an environment switch (unlike the traditional `cli` vs `fpm` distinction).
- **Output buffers**: if `ob_start()` is truly needed, wrap it in try/finally that always calls `ob_end_clean()` or `ob_get_clean()`.
- **Twig `Node::compile()` emitting `ob_start()`**: two options.
  - **Minimal patch**: emit a `try { ... } finally { $var = ob_get_clean(); }` block around the subcompiled body, so the buffer always closes even on exception.
  - **Modern refactor**: switch to the callback-based approach used by `twig/cache-extra` (`CaptureNode` + a dedicated `Runtime` service exposing `$env->getRuntime(...)`), which avoids `ob_start()` entirely and is the direction maintained by the Twig team.

### 7. Event listeners and subscribers (Warning)

**Scope:** refines §1 for the specific case of listeners and subscribers. Use this section when the stateful service is wired through `kernel.event_listener`, `kernel.event_subscriber`, or the `#[AsEventListener]` attribute.

Listeners that accumulate data across dispatches will grow unbounded in worker mode.

**Detect:**
- Properties that grow across dispatches: arrays appended to (`$this->items[] = ...`), counters incremented, maps populated without eviction
- Storage of references to request-scoped objects (`Request`, `Response`, `User`, event instances). **Why**: holding a `Request` keeps the entire previous-request object graph alive and prevents GC.
- Kernel event handlers (`onKernelRequest`, `onKernelController`, `onKernelView`, `onKernelResponse`, `onKernelTerminate`, `onKernelException`) writing to instance properties without a reset path
- Listeners on Messenger (`WorkerMessage*Event`), Security (`LoginSuccessEvent`, `AuthenticationTokenCreatedEvent`), Mailer (`MessageEvent`, `FailedMessageEvent`), or Doctrine lifecycle events (`postFlush`, `postPersist`...) accumulating data

**Ignore:**
- Listeners with only `readonly` properties or properties assigned once in `__construct()`
- Read-only listeners that only inspect the event without storing state
- Listeners that delegate to a service implementing `ResetInterface` (the service owns the state lifecycle)
- Listeners that already implement `ResetInterface` with a correct `reset()` method
- Symfony's built-in profiler/collectors: auto-reset via `kernel.reset`

**Fix:**
- **Preferred wiring**: use the `#[AsEventListener]` attribute (Symfony 6.3+) for clean auto-configuration.
- **Reset state**: implement `ResetInterface` and clear request-scoped properties in `reset()`. With autoconfigure, the `kernel.reset` tag is applied automatically; verify via `debug:container --tag=kernel.reset`.
- **Request access**: never store a `Request` on a listener property. Inject `RequestStack` and call `getCurrentRequest()` at the point of use.
- **Per-request data**: use `$request->attributes` (the attributes bag is cleared between requests) instead of instance properties.
- **Cross-request aggregation** (metrics, counters): send to a dedicated sink (Prometheus, logger, OpenTelemetry) rather than accumulating in a listener.

### 8. In-memory caches (Warning)

**Scope:** refines §1 and §2 for the specific case of keyed stores and memoization. Use this section when the stateful property (instance or static) is explicitly used as a cache (lookup by key, memoization of expensive computations).

Unbounded in-memory caches cause memory leaks over hundreds/thousands of requests.

**Detect:**
- Instance or static array properties used as caches without size limit or TTL: `$this->cache[$key] = ...`, `$this->cache[$key] ??= $this->compute($key)` (memoization decorator)
- `SplObjectStorage` used as a cache: releases entries when the key object is GC'd, but grows unbounded if keys are kept alive elsewhere
- `ArrayObject` or plain arrays used as long-lived keyed stores
- APCu used as a cache in request code: entries persist for the entire worker lifetime; without TTL or eviction they leak (see also §11)
- Doctrine caches (`QueryCache`, `ResultCache`, `MetadataCache`) configured with `ArrayAdapter` or any unbounded adapter in worker mode

**Ignore:**
- `WeakMap`: entries are garbage-collected when the key object is unreferenced elsewhere
- Bounded caches with explicit size limit, LRU eviction, or TTL
- `Symfony\Component\Cache\Adapter\ArrayAdapter` configured with `$maxItems`
- Caches wrapped in a service that implements `ResetInterface` and clears the backing store in `reset()`
- `opcache`: managed by the PHP runtime, not by user code

**Fix:**
- **Bounded in-memory cache**: `Symfony\Component\Cache\Adapter\ArrayAdapter` with `$maxItems` set (LRU eviction).
- **Shared across workers**: `RedisAdapter`, `MemcachedAdapter`, or `FilesystemAdapter` via `CacheInterface` / PSR-6.
- **Memoization**: `WeakMap` for object keys, bounded `ArrayAdapter` for scalar keys; never a raw array without a cap.
- **Per-request caches**: implement `ResetInterface` and clear the backing store in `reset()`.
- **Doctrine caches in worker mode**: configure `query_cache` and `metadata_cache` with a persistent backend (`redis`, `php_files`); avoid `ArrayAdapter` for these, or set `maxItems`.
- **APCu**: always set a TTL; prefer the `ApcuAdapter` wrapper for a consistent API.

### 9. Doctrine connection stale (Warning)

**Scope:** Doctrine DBAL and ORM connection lifecycle in worker mode. Related sections:
- Raw resource hygiene (unclosed fd, unmanaged PDO) → §5
- Persistent connections (`pg_pconnect`, PDO `ATTR_PERSISTENT`) created outside Doctrine → §5

In long-running workers, database connections may timeout and become stale ("MySQL server has gone away", "connection reset by peer").

**Detect:**
- `EntityManager` used after catching a Doctrine exception that closes it (`UniqueConstraintViolationException`, `DBALException`, etc.) without calling `ManagerRegistry::resetManager()`: a closed EM persists for the worker lifetime and rejects all future queries
- Doctrine connection or `EntityManager` held on a long-lived service property without a reset path: in Symfony 6.4+ `doctrine-bundle` auto-tags them `kernel.reset`; older versions need manual wiring
- Manual PDO/DBAL connection handling outside Doctrine's `Connection` wrapper
- Long-running code paths (batch imports, large response streams) that keep a single connection open for minutes without a reconnect strategy

**Ignore:**
- Symfony 6.4+ with a recent `doctrine-bundle`: connection and `EntityManager` auto-reset between requests (verify with `debug:container --tag=kernel.reset | grep -i doctrine`)
- Symfony `Command` classes and migration scripts: not executed in the worker HTTP loop
- Code paths that explicitly call `ManagerRegistry::resetManager()` or `Connection::close()` at the right boundary

**Fix:**
- **Symfony 6.4+**: `doctrine-bundle` auto-tags the connection and `EntityManager` as `kernel.reset`. Verify with `debug:container --tag=kernel.reset | grep -i doctrine`.
- **Older Symfony**: add a `kernel.request` listener that calls `$connection->close()` for a lazy reconnect on the next query.
- **Closed EM recovery**: after catching a Doctrine exception that closes the EM, call `$managerRegistry->resetManager()` before the next query. Without this, the worker carries a dead EM for its remaining lifetime.
- **Long-running handlers / batch imports**: periodically call `Connection::close()` or `resetManager()`. Under Messenger, wire the Doctrine ping connection middleware (`symfony/doctrine-messenger`).
- **Manual PDO**: move the connection back under Doctrine's `Connection` wrapper to inherit the reset lifecycle.
- **Diagnosis**: if "MySQL server has gone away" or "connection reset by peer" appear in prod logs, verify the reset path above and consider tuning the server-side `wait_timeout`.

### 10. Monolog buffering handlers (Warning)

Handlers that buffer log records (`BufferHandler`, `FingersCrossedHandler`, `DeduplicationHandler`) accumulate records across requests if not reset.

**Detect:**
- Monolog buffering handlers (`BufferHandler`, `FingersCrossedHandler`) instantiated manually (`new BufferHandler(...)`) or wired outside MonologBundle's DI: they do not inherit auto-tagging and need explicit `kernel.reset`
- `DeduplicationHandler` and `SamplingHandler`: keep internal state across log calls; same concern as buffering handlers when wired outside MonologBundle
- Custom handlers (extending `AbstractHandler` or implementing `HandlerInterface`) that store records in a property without implementing `Monolog\ResettableInterface`
- Custom processors (implementing `Monolog\Processor\ProcessorInterface` or callable processors) that accumulate state between log calls

**Ignore:**
- Handlers wired via MonologBundle's standard config: auto-tagged `kernel.reset` when they implement `ResettableInterface`
- Handlers or processors that already implement `Monolog\ResettableInterface` with a correct `reset()`
- Non-buffering handlers (`StreamHandler`, `SyslogHandler`, `RotatingFileHandler`, `ErrorLogHandler`): they write each record immediately, no state to leak

**Fix:**
- **Standard MonologBundle config**: no action needed if handlers or processors implement `Monolog\ResettableInterface`. Since MonologBundle 3.7.0, they are auto-tagged `kernel.reset` ([changelog][monolog-bundle-changelog]). Verify with `debug:container --tag=kernel.reset | grep -i monolog`.
- **Custom handlers**: implement `Monolog\ResettableInterface` and clear internal buffers in `reset()`.
- **Custom processors**: same rule: implement `ResettableInterface` and clear accumulated state in `reset()`.
- **Manual instantiation** (inside bundle Extensions, CompilerPass, factories): either let MonologBundle manage the handler, or manually add the `kernel.reset` tag when registering the service.

### 11. Runtime state mutations (Warning)

**Scope:** PHP runtime settings and handlers that persist across requests when mutated during request handling (this includes `putenv()`). Related sections:
- Reads from or writes to `$_ENV`, `$_SERVER`, `$GLOBALS` → §3
- `header_remove()` and raw HTTP response manipulations → §6
- `apcu_*` used as an unbounded cache → §8

**Detect:**
- Global PHP settings mutated at request time: `ini_set()`, `putenv()`, `setlocale()`, `date_default_timezone_set()`, `mb_internal_encoding()`, `mb_detect_order()`, `error_reporting()`
- Execution-control settings: `set_time_limit()`, `ignore_user_abort()` — a value set in request 1 persists for the whole worker
- GC toggling: `gc_disable()` / `gc_enable()` called in request handling without symmetric restoration
- `set_error_handler()`, `set_exception_handler()` without a matching `restore_*()` in a `finally` block
- `register_shutdown_function()` during request handling: runs at worker shutdown, not per request; per-request cleanup logic is broken
- `header_remove()` or raw HTTP response state manipulation outside Symfony's `Response` (see also §6)
- `apcu_*` used as an unbounded worker-scoped cache (see also §8)

**Ignore:**
- Settings mutated in entry-point scripts (`public/index.php`, `bin/console`, `bin/*`) and bundle boot hooks (`Bundle::boot()`, `KernelInterface::boot()`): executed once per worker start
- Settings wired through Symfony configuration (`framework.yaml`, `services.yaml`) or container parameters: applied at boot, not at request time
- Symfony `Command` classes: not executed in the worker HTTP loop

**Fix:**
- **Global settings** (`ini_set`, `setlocale`, `mb_*`, `date_default_timezone_set`, etc.): move to `public/index.php`, bundle boot, or DI config. Read once at boot, never during request handling.
- **Execution-control settings** (`set_time_limit`, `ignore_user_abort`): use Symfony's request-timeout or worker-configuration mechanisms instead; if unavoidable, restore the previous value in a `finally`.
- **Error / exception handlers**: use Symfony's `ErrorHandler` component (handles worker mode). If you must `set_error_handler()`, wrap in try/finally with `restore_error_handler()`.
- **Shutdown hooks**: use `kernel.terminate` event listener or `ResetInterface::reset()` for per-request cleanup; reserve `register_shutdown_function()` for actual worker-shutdown logic registered once at boot.
- **Raw response manipulation** (`header_remove()`): use `$response->headers->remove('X-Foo')` instead (see §6).
- **APCu as cache**: see §8 — `ApcuAdapter` via `CacheInterface` with TTL, or switch to `RedisAdapter` / `FilesystemAdapter`.

### 12. Known runtime incompatibilities (Critical / Warning / Info)

Detected primarily by scanning `composer.json` (extensions listed in `require` / `require-dev`), the project's `Dockerfile` / image base, and `php.ini` settings. Severity varies per item.

**Critical:**
- **Session data leakage ([CVE-2026-24894][cve-2026-24894])**: before FrankenPHP 1.11.2, `$_SESSION` was not reset between worker requests, exposing cross-user data. Upgrade to >= 1.11.2.
- **`ext-newrelic`**: not thread-safe, flagged as incompatible in [FrankenPHP's known issues][frankenphp-known-issues]. The New Relic PHP agent [explicitly rejects ZTS builds][newrelic-zts]. Alternative: OpenTelemetry with an OTLP exporter.
- **`ext-xdebug` enabled in production**: debugger attachment state is per-process, not per-request. Disable `zend_extension=xdebug` in prod images.

**Warning:**
- **`ext-imap`**: not thread-safe, flagged as incompatible in [FrankenPHP's known issues][frankenphp-known-issues]. Alternatives: `javanile/php-imap2`, `webklex/php-imap` (pure PHP, no extension).
- **`ext-openssl` on Alpine/musl**: [documented crashes under heavy load][frankenphp-known-issues] (musl's allocator interacts poorly with some OpenSSL codepaths). Prefer Debian-based images (`php:*-fpm`, `php:*-cli`) in production.
- **Native sessions (`session_start()` with default file handler)**: file locking across workers causes contention. Configure `framework.session.handler_id` to point to Redis or DB.
- **`pcntl_fork()`**: incompatible with worker mode by design. Use Symfony's Process component or Messenger for parallel work.

**Info:**
- **`chdir()`**: changes the cwd for the entire worker and all subsequent requests. Use absolute paths instead.
- **OPcache `validate_timestamps`**: in production, set to `0` to avoid file-stat overhead; in dev, keep `1`. A worker restart is required to pick up code changes when `0`.
- **OPcache preloading (`opcache.preload`)**: classes preloaded at worker startup; changes require a worker restart.

## Report format

Output a report titled "FrankenPHP Worker Mode Compatibility Report" with:
- **Path** and **files scanned** count
- Findings grouped by severity: **Critical**, **Warning**, **Info**
- Each finding: category, file:line, what was detected, risk, and fix
- A short summary with count per severity and top priority actions

Use standard severity levels (Critical, Warning, Info) so that any formatting rule can apply its own visual indicators.

If no issues are found, state that the code appears compatible and note any assumptions made.

## Technologies covered

| Technology | Reference version |
|---|---|
| FrankenPHP | worker mode |
| Symfony | 6.4 LTS / 7.4 LTS / 8.0 |
| PHP | 8.2 / 8.3 / 8.4 |

## Reference sources

**FrankenPHP**
- [Worker mode docs][frankenphp-worker]
- [Known issues][frankenphp-known-issues]
- [CVE-2026-24894: session data leakage][cve-2026-24894]

**Symfony**
- [Resetting services][symfony-reset]
- [`kernel.reset` tag reference][symfony-kernel-reset]
- [Symfony 7.4: native FrankenPHP runtime support][symfony-7.4-release]
- [MonologBundle CHANGELOG][monolog-bundle-changelog]

**Twig**
- [Legacy `twig/twig-cache-extension` CacheNode (unsafe `ob_start` pattern)][twig-cache-extension-node]
- [Modern `twig/cache-extra` CacheNode (callback-based, avoids `ob_start`)][twig-cache-extra-node]

**External**
- [New Relic PHP agent: ZTS not supported][newrelic-zts]

[frankenphp-worker]: https://frankenphp.dev/docs/worker/
[frankenphp-known-issues]: https://frankenphp.dev/docs/known-issues/
[symfony-reset]: https://symfony.com/doc/current/service_container/reset.html
[symfony-kernel-reset]: https://symfony.com/doc/current/reference/dic_tags.html#kernel-reset
[symfony-7.4-release]: https://symfony.com/blog/new-in-symfony-7-4-dx-improvements-part-2
[monolog-bundle-changelog]: https://github.com/symfony/monolog-bundle/blob/master/CHANGELOG.md
[newrelic-zts]: https://docs.newrelic.com/docs/apm/agents/php-agent/getting-started/php-agent-compatibility-requirements/
[cve-2026-24894]: https://github.com/php/frankenphp/security/advisories/GHSA-r3xh-3r3w-47gp
[twig-cache-extension-node]: https://github.com/twigphp/twig-cache-extension/blob/master/lib/Node/CacheNode.php
[twig-cache-extra-node]: https://github.com/twigphp/cache-extra/blob/3.x/Node/CacheNode.php
