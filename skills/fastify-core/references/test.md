# Fastify Core Testing & Type Testing (`test`)

Load this reference when writing, debugging, or reviewing runtime unit tests (`test/**/*.test.js`), bundler tests (`test/bundler/`), production integration tests (`integration/`), or TypeScript type tests (`test/types/*.tst.ts`) in `fastify/fastify`.

---

## 1. Runtime Unit Tests (`borp` + `node:test`)

### Running Unit Tests
- **Single test file (fast iteration):**
  ```bash
  npx borp -p test/hooks.test.js
  ```
- **Full unit test suite:**
  ```bash
  npm run unit
  ```
- **100% Coverage Check (matches CI `coverage-nix` / `coverage-win`):**
  ```bash
  npm run coverage:ci-check-coverage
  ```
- `.borp.yaml` runs all files matching `test/**/*.test.js` and `test/**/*.test.mjs`.

### Writing Unit Tests in `fastify/fastify`
1. **Test Runner & Assertions:**
   - Import `{ test, describe }` from `'node:test'`.
   - Either use `t.plan(N)` + `t.assert.*` (`t.assert.strictEqual`, `t.assert.deepStrictEqual`, `t.assert.ifError`, `t.assert.ok`, `t.assert.rejects`) OR `const { tspl } = require('@matteo.collina/tspl')`.
   - **Critical `tspl` Rule:** Whenever a test uses `const p = tspl(t, { plan: N })` (or destructured `{ strictEqual, ok, completed } = tspl(t, { plan: N })`), you **must** `await p.completed` (or `await completed`) at the end of the `async t => { ... }` function so callback assertions finish before the test exits:
     ```js
     'use strict'

     const { test } = require('node:test')
     const { tspl } = require('@matteo.collina/tspl')
     const Fastify = require('..')

     test('hook execution order', async t => {
       const p = tspl(t, { plan: 2 })
       const fastify = Fastify()
       t.after(() => fastify.close())

       fastify.addHook('onRequest', (req, reply, done) => {
         p.ok(req)
         done()
       })

       fastify.get('/', async () => ({ hello: 'world' }))

       const res = await fastify.inject({ method: 'GET', url: '/' })
       p.strictEqual(res.statusCode, 200)
       await p.completed
     })
     ```
2. **In-Memory `fastify.inject()` vs Real Listening Server (`fastify.listen({ port: 0 })`):**
   - **Use `await fastify.inject(...)` by default** for routing, hooks, decorators, schema validation, serialization, content-type parsing, and error handling.
   - **Use `await fastify.listen({ port: 0 })` + `t.after(() => fastify.close())`** when testing real TCP/HTTP sockets, `connectionTimeout`, `keepAliveTimeout`, `forceCloseConnections`, `maxRequestsPerSocket`, `clientErrorHandler`, `onRequestAbort`, stream premature close, HTTP/2 (`test/http2/`), or HTTPS (`test/https/`).
   - When making real HTTP requests against a listening server in `fastify/fastify` tests, use `undici` (`const { Client, request } = require('undici')`) or `node:http` / `h2url` (for HTTP/2), and always close the client/agent and `fastify.close()` in `t.after()`.
3. **Testing Internal Modules Directly (`test/internals/` & `kTestInternals`):**
   - `test/internals/*.test.js` unit-tests internal helpers in `lib/` (such as `handle-request.test.js`, `reply.test.js`, `request.test.js`, `hooks.test.js`, `content-type-parser.test.js`, `logger.test.js`, `initialConfig.test.js`).
   - Some `lib/` modules expose internal functions exclusively for testing under `[kTestInternals]` (`const { kTestInternals } = require('./lib/symbols')`).

---

## 2. Type Tests (`test/types/*.tst.ts` with TSTyche)

### Running Type Tests
- **Single type test file (fast iteration):**
  ```bash
  npx tstyche test/types/hooks.tst.ts
  ```
- **Full type test suite:**
  ```bash
  npm run test:types
  ```

### Organization of `test/types/`
`fastify/fastify` organizes TSTyche type tests by domain inside `test/types/` (using `test/types/tsconfig.json` which extends `fastify-tsconfig`):
- `test/types/fastify.tst.ts` — Top-level `fastify()` overloads (`http`, `https`, `http2`), options, and exports.
- `test/types/instance.tst.ts` — `FastifyInstance` methods, decorators, schemas, and lifecycle methods.
- `test/types/route.tst.ts` — Route shorthands, `RouteGenericInterface`, `ContextConfig`, and handler inference.
- `test/types/hooks.tst.ts` — All application and request/reply hook signatures (sync, async, callback).
- `test/types/request.tst.ts` & `test/types/reply.tst.ts` — `FastifyRequest` and `FastifyReply` properties/methods.
- `test/types/type-provider.tst.ts` — `withTypeProvider<...>()` generic narrowing (`json-schema-to-ts`, `typebox`).
- `test/types/register.tst.ts` & `test/types/plugin.tst.ts` — `fastify.register()` options and plugin callbacks/async plugins.
- `test/types/errors.tst.ts` — `FastifyError` and `errorCodes.FST_ERR_*`.
- `test/types/logger.tst.ts`, `test/types/schema.tst.ts`, `test/types/content-type-parser.tst.ts`, `test/types/decorate-request-reply.tst.ts`, `test/types/serverFactory.tst.ts`, `test/types/using.tst.ts`.

### Authoring TSTyche Assertions in `test/types/*.tst.ts`
- Import `{ expect }` (and `{ test, describe }` when grouping) from `'tstyche'`.
- Always include **both positive assertions** (`expect(val).type.toBe<Expected>()`, `expect(val).type.toBeAssignableTo<Expected>()`) and **negative assertions** (`expect(fn).type.not.toBeCallableWith(...)`) for new options, overloads, or generics.
- Avoid deprecated `.toRaiseError()` in new assertions; use `.not.toBeCallableWith(...)` or specific ability matchers.
- Ensure `test/types/*.tst.ts` files pass `npm run lint` (`max-len: 120`, `comma-dangle: never`).

---

## 3. Bundler & Production Integration Tests

1. **Bundler Tests (`test/bundler/webpack` & `test/bundler/esbuild`):**
   - Verifies that `fastify` can be bundled with Webpack and esbuild without breaking generated validators or dynamic requires (`ci.yml` `package` job):
     ```bash
     (cd test/bundler/webpack && npm install --ignore-scripts && npm test)
     (cd test/bundler/esbuild && npm install --ignore-scripts && npm test)
     ```
2. **Production Smoke Test (`integration/`):**
   - `integration/server.js` and `integration/test.sh` verify that Fastify starts and serves routes when installed with production-only dependencies (`pnpm install --ignore-scripts --prod`).

Run both bundler commands from the repository root; the subshells preserve that working directory. During review, execute install/build checks in the isolated checkout when they can modify tracked files.

---

## 4. Regression Scenarios for Lifecycle & Scope Changes

Select scenarios for the affected contract rather than adding this entire matrix to every PR:

| Changed behavior | Regression evidence |
| --- | --- |
| Early reply from a hook | Assert the response and that subsequent hooks/handler do not execute; cover callback or async style as affected. |
| Error propagation | Assert which scoped error handler receives the error and that a response is sent once; verify `onError` side effects without sending from that hook. |
| Response transformations | Exercise object payloads and relevant string/Buffer/stream/null cases to distinguish `preSerialization` from `onSend`. |
| Encapsulation | Check parent-to-child inheritance and child/sibling isolation for affected decorators, hooks, schemas, or error handlers. Include the intentional shared behavior of `fastify-plugin` where relevant. |
| Per-instance or per-request state | Run two instances/registrations with different options and concurrent requests where relevant; assert no shared mutable state. |
| Validator/serializer changes | Test the result contract, error paths, and response schema selection described in [best-practices.md](best-practices.md). |
| Type-provider changes | Add type assertions in the actual plugin scope; verify provider inference and runtime schema behavior separately. |

Use existing tests and public behavior as the oracle. A focused regression should fail without the fix and pass with it; retain the repository's configured coverage gates.
