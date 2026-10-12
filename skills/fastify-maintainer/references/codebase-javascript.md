# Codebase Architecture & JavaScript Guidelines

Load this reference when reviewing or changing runtime JavaScript (`.js`, `.cjs`, `.mjs`), plugin architecture, hooks, lifecycle, encapsulation, error handling, or performance-sensitive code across Fastify repositories.

## 1. Codebase Architecture & Fastify Lifecycle

### Encapsulation & `fastify-plugin` (`fp`)
- **Encapsulation by Default:** Fastify creates a new encapsulated context on `fastify.register()`. Decorators, hooks, schemas, and content-type parsers registered inside a plugin are visible **only** to that plugin and its descendants unless wrapped with `fastify-plugin` (`fp`) or marked with `Symbol.for('skip-override')`.
- **Official Ecosystem Plugins:**
  - Plugins that expose decorators (`decorate`, `decorateRequest`, `decorateReply`) or hooks intended for parent scopes **must** wrap their export with `fp(pluginFn, { fastify: '5.x', name: '@fastify/...', dependencies: [...], decorators: { ... } })`.
  - Plugins that encapsulate routes (e.g., `fastify-swagger-ui`) should **not** break encapsulation on their route-registration sub-plugins unless explicitly intended.
- **Decorator Pre-declaration & Shape Stability:**
  - Always initialize request/reply reference-type decorators using `{ getter, setter }` or `null` at registration time (`fastify.decorateRequest('user', null)`) and populate them inside hooks (`onRequest`, `preHandler`). Never pass a shared mutable object or array directly to `decorateRequest('items', [])` (which shares state across all concurrent requests!).

### Hook & Plugin Completion Rules
- **Never Mix `async`/`Promise` and `done` Callbacks:**
  - If a plugin function, hook (`onRequest`, `preHandler`, `onSend`, `onResponse`, `onError`, `onClose`), or custom parser is `async` or returns a `Promise`, it **must not** call `done()`. Calling `done()` inside an `async` function causes double-execution errors (`FST_ERR_HOOK_INVALID_ASYNC_HANDLER` or `avvio` double callback) or uncaught rejections.
  - In callback-style hooks/plugins (`function (req, reply, done)`), ensure **every** code path—including early returns and error branches—calls `done()` or `done(err)` exactly once (or calls `reply.send()` in request/reply hooks where applicable).
- **Resource Teardown (`onClose`):**
  - Any plugin that opens connections, timers, worker threads, or file handles (`@fastify/redis`, `@fastify/postgres`, `@fastify/mongodb`, `@fastify/schedule`, `@fastify/under-pressure`) must register an `onClose` hook to clean up resources when `fastify.close()` runs.

### Generated Files & Package Boundaries
- **Fastify Core (`fastify/fastify`):**
  - `lib/config-validator.js` and `lib/error-serializer.js` are generated via `build/build-validation.js` and `build/build-error-serializer.js`. Never hand-edit generated files without updating the generator and running `npm run build:validation` + `npm run test:validator:integrity`.
- **Export Maps & Module Format:**
  - Most Fastify repositories use CommonJS (`'use strict'` at the top of `.js` files where not exempted by modern `neostandard`). Preserve the repository's existing `module.exports` and ESM wrapper patterns (`package.json#exports`).

---

## 2. JavaScript Runtime Semantics, Errors, Security & Performance

### Standardized Errors (`@fastify/error`) & Deprecations (`process-warning`)
- **Error Definitions (`@fastify/error`):**
  - Do not throw ad-hoc `new Error('...')` for public or user-triggerable error paths if the repository uses `@fastify/error` (`lib/errors.js` or `errors.js`).
  - Define static error factories at module load time with uppercase `FST_ERR_<COMPONENT>_<REASON>` codes, explicit message templates (`%s`), and appropriate HTTP `statusCode` when applicable:
    ```js
    const createError = require('@fastify/error')
    const FST_ERR_MY_PLUGIN_INVALID_OPT = createError(
      'FST_ERR_MY_PLUGIN_INVALID_OPT',
      'Invalid option "%s" provided',
      500,
      TypeError
    )
    ```
- **Deprecation Warnings (`process-warning`):**
  - When deprecating an option or API, use `process-warning` (`lib/warnings.js`) with a unique `FSTDEP*` code and test that the warning is emitted once.

### Hot-Path Performance Discipline
Fastify's core value proposition is low overhead. Inspect request-time code (`onRequest`, routing, parsing, validation, serialization, `onSend`) for avoidable overhead:
1. **Move Work to Registration / Startup Time:**
   - Validate plugin options, compile schemas/regexes, pre-compute header strings, and bind closures during plugin registration (`register`) or route declaration (`onRoute`), **not** inside per-request handlers.
2. **Avoid Unnecessary Closures & Allocations Per Request:**
   - Hoist helper functions to module scope or class prototypes when possible instead of allocating new arrow functions per request.
   - Maintain monomorphic object shapes (initialize all properties on `Request` / `Reply` / context objects consistently in constructors or decorators rather than adding properties dynamically with `delete`).
3. **Avoid `async/await` Overhead in Synchronous Fast Paths:**
   - Do not convert synchronous hot-path functions into `async` functions unnecessarily (which forces Promise allocation and microtask scheduling on every request).
4. **Justified Benchmarks:**
   - When a change claims a performance improvement or touches a critical hot path in `fastify/fastify`, `fast-json-stringify`, or `fast-uri`, verify using the repository's benchmark script (`npm run bench` or `npm run benchmark`) when feasible.

### Security & Input Boundaries
- **Prototype Pollution:**
  - Use `Object.create(null)` or `Map` for lookup tables keyed by user input (headers, query params, cookie names, route params). Never index plain objects `{}` directly with unvalidated keys (`obj[userKey]`) without `Object.hasOwn(obj, userKey)` and blocking `__proto__`, `constructor`, and `prototype`.
- **Regular Expression Denial of Service (ReDoS):**
  - Avoid unbounded nested quantifiers on untrusted input headers, URLs, or payloads; consider `safe-regex2` or linear-time string parsing where appropriate.
- **Stream & Payload Safety:**
  - Respect `bodyLimit`, max header/field sizes, and always attach `'error'` and `'close'` / `stream.pipeline` handlers when piping or consuming Node.js streams so aborted client connections cannot crash the process with an unhandled `'error'` event.
