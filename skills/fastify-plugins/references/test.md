# Fastify Plugin Runtime & Integration Testing Patterns (`test`)

Load this reference when writing, reviewing, or debugging unit tests, coverage checks, or integration tests (`test/**/*.test.js`) in a Fastify plugin repository.

For standalone utilities, use the existing runner and exercise exported functions/classes directly. The request, registration, and teardown examples below apply only when the package's behavior involves Fastify. Runtime changes need relevant tests; type-only and documentation changes need validation appropriate to their scope.

---

## 1. Test Runners & 100% Coverage Enforcement

Common plugin setups use Node's built-in test runner (`node:test`) with coverage gates via `c8` or `borp`. Confirm the actual runner, metrics, and thresholds in the checkout:

1. **`c8 --100 node --test` (Most Common):**
   - Used in `@fastify/cookie`, `@fastify/multipart`, `@fastify/cors`, `@fastify/jwt`, `@fastify/rate-limit`, `@fastify/redis`, `@fastify/postgres`, `@fastify/swagger`, `@fastify/bearer-auth`, `@fastify/helmet`, `@fastify/websocket`, `@fastify/session`, `@fastify/sse`, `@fastify/otel`, `@fastify/csrf-protection`, `fastify-plugin`.
   - Run a single test file quickly during development:
     ```bash
     node --test test/my-feature.test.js
     ```
   - Run the full unit test suite with 100% coverage gate:
     ```bash
     npm run test:unit   # or `npm run unit` in @fastify/redis
     ```
2. **`borp -C --check-coverage`:**
   - Used in `@fastify/sensible`, `@fastify/static`, `@fastify/under-pressure`, `@fastify/autoload`.
   - Configured via `package.json#scripts` (`borp -C --check-coverage --lines 100 --reporter=@jsumners/line-reporter`) and optional `.borp.yaml`.
   - Run a single test file quickly:
     ```bash
     npx borp -p test/my-feature.test.js
     ```

> **Coverage Evidence:** Preserve the configured thresholds and cover changed behavior, including errors and defaults. A `--lines 100` gate establishes a line threshold; do not infer 100% branch/function thresholds from it. Report the actual coverage command and metrics separately from ordinary unit-test success.

---

## 2. `node:test` Structure & Assertion Patterns

### Pattern A: `node:test` with Built-in `t.plan(N)` & `t.assert.*` (Standard in Plugins)
Most Fastify plugin test suites use `node:test`'s built-in `t.plan(N)` and `t.assert.*` methods (`strictEqual`, `deepStrictEqual`, `ok`, `ifError`, `rejects`, `throws`) without external assertion libraries:

```js
'use strict'

const { test, describe } = require('node:test')
const Fastify = require('fastify')
const myPlugin = require('../index')

describe('myPlugin feature', () => {
  test('registers decorator and handles request via inject', async (t) => {
    t.plan(3)

    const fastify = Fastify()
    t.after(() => fastify.close())

    await fastify.register(myPlugin, { someOption: 'value' })

    fastify.get('/test', async (req, reply) => {
      t.assert.strictEqual(req.myData, 'value')
      return { ok: true }
    })

    const res = await fastify.inject({
      method: 'GET',
      url: '/test'
    })

    t.assert.strictEqual(res.statusCode, 200)
    t.assert.deepStrictEqual(res.json(), { ok: true })
  })

  test('rejects invalid option at registration', async (t) => {
    t.plan(1)

    const fastify = Fastify()
    t.after(() => fastify.close())

    await t.assert.rejects(
      fastify.register(myPlugin, { hook: 'invalidHookName' }),
      /Invalid value provided/
    )
  })
})
```

### Pattern B: `node:test` with `@matteo.collina/tspl`
When a plugin repository uses `@matteo.collina/tspl` (or when testing callback/event-driven flows across multiple asynchronous ticks):
- Create the plan with `const p = tspl(t, { plan: N })`.
- **Always** `await p.completed` at the end of the `async (t) => { ... }` test function so `node:test` waits for all planned callback assertions before finishing.

### Pattern C: Callback-Style `test('...', (t, done) => { ... })`
For tests verifying callback-style `fastify.ready(err => { ...; done() })`:
- Always call `t.plan(N)` at the start of the test and invoke `done()` at the end of the callback.

---

## 3. In-Process `fastify.inject()` vs Real Network / Service Integration Tests

Choose the smallest, fastest test boundary that accurately tests the plugin's behavior:

| Plugin Behavior Under Test | Required Test Mechanism | Key Rules |
| --- | --- | --- |
| **Decorators, hooks (`onRequest`, `preParsing`, `preValidation`, `preHandler`, `onSend`), headers, cookies, status codes, schemas, serialization, encapsulation** | **In-process `await fastify.inject(...)`** | Default for >90% of plugin tests. Exercises the complete Fastify lifecycle in memory via `light-my-request` without binding TCP ports. |
| **Stream aborts, premature client disconnect, multipart file streaming limits, WebSockets, SSE, HTTP/2, keep-alive, or backpressure (`@fastify/multipart`, `@fastify/websocket`, `@fastify/sse`, `@fastify/http-proxy`, `@fastify/reply-from`)** | **Ephemeral Server (`await fastify.listen({ port: 0 })`)** | `fastify.inject()` buffers payloads in memory and cannot simulate mid-stream TCP resets (`socket.destroy()`), HTTP `Upgrade` handshakes, or OS socket backpressure. Always use `{ port: 0 }` and `t.after(() => fastify.close())`. |
| **External Backing Services (`@fastify/redis`, `@fastify/postgres`, `@fastify/mysql`, `@fastify/mongodb`, `@fastify/kafka`, `@fastify/elasticsearch`)** | **Real Driver + Service Container** | Match the service ports and credentials configured in `.github/workflows/ci.yml` (`fastify/workflows/plugins-ci-<service>.yml`) and `package.json` helper scripts (e.g., `npm run redis`). Clean up keys/tables and `await fastify.close()` in `t.after()`. |
| **Dual CJS/ESM TypeScript Plugins (`@fastify/type-provider-typebox`, `@fastify/type-provider-json-schema-to-ts`)** | **Dual Entry-Point Runner (`test/index.cjs` & `test/index.mjs`)** | Run shared test suites against both compiled `dist/cjs/index.js` and `dist/esm/index.mjs` (`npm run test:node`). |

---

## 4. Resource Cleanup & Anti-Flakiness Checklist

1. **Always Close Fastify Instances (`t.after(() => fastify.close())`):**
   - Especially when a plugin registers timers (`@fastify/under-pressure`, `@fastify/rate-limit`), file streams (`@fastify/multipart`, `@fastify/static`), or database clients (`@fastify/redis`, `@fastify/postgres`), failing to close `fastify` leaves handles open and hangs `node --test`.
2. **Never Hardcode Listening Ports:**
   - When `fastify.listen` is needed, always use `await fastify.listen({ port: 0 })` and inspect `fastify.listeningOrigin` or `fastify.server.address().port`.
3. **Clean Up Temporary Files & Fake Timers:**
   - If a test writes uploaded files to disk (`@fastify/multipart`) or installs `@sinonjs/fake-timers` / `sinon.spy`, restore clocks and `unlink` temporary files inside `t.after()`.
4. **Honest Reporting When Docker Services Are Offline:**
   - If working in an environment without Docker or a running Redis/Postgres/Mongo/MySQL/Kafka/Elasticsearch instance, run `npm run lint` and `npm run test:typescript` (`tstyche`), and explicitly report which service-dependent runtime tests could not execute locally.
