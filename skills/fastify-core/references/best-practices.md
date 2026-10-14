# Fastify Core Best Practices, Performance, TypeScript & Docs (`best-practices`)

Load this reference when working on hot-path code (`lib/handle-request.js`, `lib/request.js`, `lib/reply.js`, `lib/hooks.js`, `lib/route.js`), TypeScript definitions (`fastify.d.ts`, `types/*.d.ts`), code style (`eslint.config.js`), or documentation (`docs/`).

---

## 1. V8 Monomorphism & Object Shape Stability

Fastify's performance depends on keeping `Request` and `Reply` instances monomorphic (sharing a single V8 hidden class / shape across requests):

1. **Pre-define All Properties in Constructors / Prototypes:**
   - Every property on `Request` (`lib/request.js`) and `Reply` (`lib/reply.js`) must be initialized in the constructor or defined on the prototype (`null`, `undefined`, `false`, or a symbol key from `lib/symbols.js`).
   - Never add ad-hoc properties to `req` or `reply` midway through request processing that were not declared on the constructor or prototype, and **never** use `delete req.prop` or `delete reply.prop`.
2. **Shape Verification (`test/same-shape.test.js`):**
   - `test/same-shape.test.js` verifies that two consecutive requests through `fastify.inject()` produce `Request` and `Reply` objects with identical V8 shapes even when decorators are populated conditionally in hooks.
   - Whenever you touch `lib/request.js`, `lib/reply.js`, or `lib/decorate.js`, run:
     ```bash
     npx borp -p test/same-shape.test.js
     ```

---

## 2. Request Lifecycle & Hot-Path Performance Discipline

1. **Compile at Registration / Route Declaration Time, Not Request Time:**
   - Pre-compute route context properties, hook runner functions, schema validators, and serializers inside `lib/route.js` or plugin registration.
   - Check whether a hook array is empty (`context.onRequest !== null`) at route setup so routes with no hooks skip the hook runner entirely during request execution.
2. **Avoid Mixing `async` and `done` Callbacks:**
   - Plugin registration that mixes async and callback styles can emit `FSTWRN003`. Async hooks with an invalid callback-style signature are rejected with `FST_ERR_HOOK_INVALID_ASYNC_HANDLER`; do not apply the plugin warning to hooks or route handlers.
   - In core hook runners (`lib/hooks.js`), handle Promise returns (`then(next, done)`) and synchronous `done()` callbacks on strictly separate branches so `next()` / `done()` can never be invoked twice.
3. **Avoid Unnecessary Closures and Allocations in `lib/handle-request.js` & `lib/reply.js`:**
   - Use named module-level functions and pass context/state as parameters rather than allocating closures per request.
   - Avoid `async`/`await` inside synchronous fast paths in `lib/handle-request.js` and `lib/reply.js`; use `wrapThenable` (`lib/wrap-thenable.js`) only when the user handler actually returns a thenable (`payload && typeof payload.then === 'function'`).
4. **Benchmarking Hot-Path Changes:**
   - When modifying `lib/handle-request.js`, `lib/reply.js`, `lib/request.js`, `lib/hooks.js`, or `lib/content-type-parser.js`, compare throughput when appropriate using `npm run benchmark` or `npm run bench` (`branchcmp`).

---

## 3. Core ESLint & Code Style Rules (`eslint.config.js`)

`fastify/fastify` uses `neostandard` (`ts: true`) with three stricter custom rules in `eslint.config.js`:

1. **No Trailing Commas (`comma-dangle: ['error', 'never']`):**
   - Even in multiline objects, arrays, imports, and TypeScript interfaces/type parameters.
2. **120-Character Line Length Limit (`max-len: ['error', { code: 120, ... }]`):**
   - Applies across `.js`, `.d.ts`, and `.tst.ts` files (ignoring long URLs, strings, template literals, regexes, and comments). Wrap long TypeScript generic signatures across multiple lines to stay under 120 characters.
3. **2-Space Binary Operator Indentation in `.d.ts` (`@stylistic/indent-binary-ops: ['error', 2]`):**
   - In `fastify.d.ts` and `types/*.d.ts`, multiline union (`|`) and intersection (`&`) types must use 2-space indentation.
4. **`'use strict'` & `node:` Prefix:**
   - Every `.js` file begins with `'use strict'` and imports Node built-ins using the `node:` protocol (`require('node:http')`, `require('node:stream')`, `require('node:test')`).

Always run `npm run lint` after editing any `.js`, `.d.ts`, or `.tst.ts` file.

---

## 4. Core TypeScript Architecture (`fastify.d.ts` & `types/*.d.ts`)

Unlike plugins that have a single `types/index.d.ts`, `fastify/fastify` splits its declarations across `fastify.d.ts` (root entry) and modular files in `types/*.d.ts`. Inspect the checkout for the current module list:

1. **Preserve the 5 Core Instance Generics:**
   `FastifyInstance` and `fastify()` factory overloads carry 5 ordered type parameters with defaults:
   ```ts
   FastifyInstance<
     RawServer extends RawServerBase = RawServerDefault,
     RawRequest extends RawRequestDefaultExpression<RawServer> = RawRequestDefaultExpression<RawServer>,
     RawReply extends RawReplyDefaultExpression<RawServer> = RawReplyDefaultExpression<RawServer>,
     Logger extends FastifyBaseLogger = FastifyBaseLogger,
     TypeProvider extends FastifyTypeProvider = FastifyTypeProviderDefault
   >
   ```
2. **Preserve Route & Hook Generic Propagation:**
   Route methods (`route`, `get`, `post`, etc. in `types/route.d.ts`) and hook handlers (`types/hooks.d.ts`) propagate:
   - `RouteGeneric extends RouteGenericInterface = RouteGenericInterface`
   - `ContextConfig = ContextConfigDefault`
   - `SchemaCompiler extends FastifySchema = FastifySchema`
   - `TypeProvider extends FastifyTypeProvider = FastifyTypeProviderDefault`
   - `Logger extends FastifyBaseLogger = FastifyBaseLogger`
   Never drop or hardcode these generics to `any` or defaults when adding method overloads.
3. **Export Parity in `fastify.d.ts`:**
   When adding a new public type in `types/*.d.ts`, re-export it in `fastify.d.ts` under `declare namespace fastify { export type { ... } }` so consumers can import it directly from `'fastify'`.
4. **Type-Provider Scope:**
   Provider inference does not propagate globally into encapsulated plugins. Test inference inside the affected registration scope and preserve its provider generic or explicit `withTypeProvider<...>()` call; global declaration augmentation does not prove that a decorator exists in every runtime scope.

---

## Schema Validation & Serialization Contracts

Apply these checks when changing schema compilation, validation, serialization, or their hooks:

- Treat schemas passed to compilers as trusted application code: default compilation uses `new Function()`. Do not introduce compilation of request-supplied schemas.
- Keep database and other asynchronous business checks after initial validation, typically in `preHandler`, rather than adding them to initial schema validation.
- Synchronous custom validators use `{ value }` for transformed data or `{ error }` for failure. Test their interaction with async `preValidation` hooks without throwing from the synchronous validator.
- Asynchronous validators signal failure by rejecting. Their resolved data is not a synchronous `{ value, error }` result envelope. Test payloads containing literal `value` and `error` keys so they cannot replace request data or manufacture validation failures.
- Test the selected response schema for the affected status/content type, including whether sensitive fields are excluded under its `additionalProperties` policy. Preserve custom serializer behavior; a response schema is not a universal output-validation guarantee.

Behavior references checked on 2026-10-14 against Fastify revision `e222f53`: [Hooks](https://github.com/fastify/fastify/blob/e222f53aa267080b349eacdeaff03ea9a061e8c7/docs/Reference/Hooks.md), [Warnings](https://github.com/fastify/fastify/blob/e222f53aa267080b349eacdeaff03ea9a061e8c7/docs/Reference/Warnings.md), [Errors](https://github.com/fastify/fastify/blob/e222f53aa267080b349eacdeaff03ea9a061e8c7/docs/Reference/Errors.md), [Validation and Serialization](https://github.com/fastify/fastify/blob/e222f53aa267080b349eacdeaff03ea9a061e8c7/docs/Reference/Validation-and-Serialization.md), and [Type Providers](https://github.com/fastify/fastify/blob/e222f53aa267080b349eacdeaff03ea9a061e8c7/docs/Reference/Type-Providers.md). Resolve version-specific differences from the target branch's docs and tests.

---

## 5. Documentation & Style Guide Rules (`docs/`)

1. **Keep Reference Docs in Lockstep:**
   - Server options -> `docs/Reference/Server.md`
   - Route options -> `docs/Reference/Routes.md`
   - Hooks -> `docs/Reference/Hooks.md`
   - Request / Reply methods & properties -> `docs/Reference/Request.md` / `docs/Reference/Reply.md`
   - Error codes (`FST_ERR_*`) -> `docs/Reference/Errors.md`
   - Warning / deprecation codes (`FSTWRN*`, `FSTSEC*`, `FSTDEP*`) -> `docs/Reference/Warnings.md`
2. **Fastify Documentation Style Guide (`docs/Guides/Style-Guide.md`):**
   - **Pronouns:** Use second person ("you") in `docs/Guides/*.md`, and formal third person (avoid "you") in `docs/Reference/*.md`.
   - **No Images or Videos:** Never embed images or videos in `docs/`.
   - **80-Column / Markdown Linting:** Run `npm run lint:markdown` (`markdownlint-cli2`, configured via `.markdownlint-cli2.yaml`) whenever any `.md` file is modified.
3. **`docs/Guides/Ecosystem.md` Alphabetical Ordering:**
   - Plugins listed in `docs/Guides/Ecosystem.md` must be strictly alphabetical by name (`lint-ecosystem-order.yml` runs `.github/scripts/lint-ecosystem.js` on PRs touching `Ecosystem.md`).
