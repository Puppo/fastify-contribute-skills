# Runtime & Integration Testing Guidelines

Load this reference when reproducing bugs, writing or reviewing unit/regression tests, or deciding whether integration tests (real sockets, external services, or package-consumer tests) are required.

## 1. Runtime Test Runners & Idioms Across Fastify

Always inspect `package.json#scripts` and existing `test/*.test.js` files before writing tests. Fastify repositories use three main runtime test setups:

### A. `node:test` (or `borp`) + `@matteo.collina/tspl` / `node:assert` (Current Standard)
Used by `fastify/fastify`, `fastify-sensible`, `fastify-multipart`, `fastify-cookie`, `middie`, `under-pressure`, `fastify-plugin`, `avvio`, `light-my-request`, `fastify-error`, `fast-json-stringify`, and most active ecosystem plugins.
- **Targeted Execution:**
  - If using `borp`: `npx borp -p test/my-feature.test.js`
  - If using `node --test`: `node --test test/my-feature.test.js`
- **`@matteo.collina/tspl` Plan Completion Rule:**
  - When a test uses `tspl(t, { plan: N })` with callbacks, event emitters, or streams, you **must** `await p.completed` (or `await t.completed`) at the end of the `async (t) => { ... }` function. Without `await p.completed`, `node:test` finishes the async function early and fails with a premature plan mismatch or unhandled error:
    ```js
    const { test } = require('node:test')
    const { tspl } = require('@matteo.collina/tspl')
    const Fastify = require('fastify')

    test('handles custom hook callback', async t => {
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

### B. `tape` + `playwright-test` (e.g., `fast-uri`)
- Run unit tests via `npm run test:unit` (`tape test/**/*.js`) and browser compatibility tests via `npm run test:browser` when available.

### C. Legacy `tap` (Maintenance Branches or Older Suites)
- Uses `t.plan(N)`, `t.teardown(() => fastify.close())`, and `t.equal` / `t.same`. Do not introduce `tap` into repositories that have already migrated to `node:test` / `borp`.

### Coverage Thresholds
- Many Fastify repositories enforce **100% line/branch coverage** (`c8 --100 node --test` or `borp --coverage --check-coverage --lines 100`).
- Every bug fix or feature must test both the happy path and error/validation branches so the full `npm run test:unit` / `npm run unit` coverage check passes.

---

## 2. Integration Test Decision Criteria (`REQ-006`)

Always record an explicit **Integration Test Required / Not Required** decision with rationale. Choose the smallest test boundary that exercises the changed behavior:

| Boundary Under Test | Required Test Mechanism | Rationale & Rules |
| --- | --- | --- |
| **Routes, hooks, decorators, schemas, serialization, headers, cookies, encapsulation** | **In-process `fastify.inject()`** (`light-my-request`) — *No network integration test required* | `fastify.inject()` exercises the full Fastify request/reply lifecycle, hooks, parsers, and serializers in memory without socket overhead or port flakiness. |
| **Raw TCP/HTTP sockets, `keep-alive`, pipelining, client abort/disconnect, stream truncation, `onTimeout`, WebSockets, SSE, HTTP proxying** | **Real Ephemeral Server (`await fastify.listen({ port: 0 })`)** — *Network integration test required* | In-memory injection does not simulate real OS socket backpressure, half-closed TCP connections, `socket.destroy()`, or HTTP upgrade handshakes. Always bind `{ port: 0 }` (never hardcode ports) and register `t.after(() => fastify.close())`. |
| **Database, cache, or message-broker plugins (`@fastify/redis`, `@fastify/postgres`, `@fastify/mysql`, `@fastify/mongodb`, `@fastify/kafka`, `@fastify/elasticsearch`)** | **External Service Integration Test** — *Service fixture required* | Verify connection pooling, decorator attachment, error propagation, and `onClose` teardown against the real driver/service defined in `.github/workflows/*.yml` or `docker-compose.yml`. Isolate keys/tables per test. If the service container is not running locally, state that clearly in the validation report instead of faking a pass. |
| **Dual CJS/ESM package exports or CLI binaries (`fastify-type-provider-typebox`, `fastify-cli`)** | **Package / Subprocess Integration Test** | Test both `.cjs` and `.mjs` entry points (`test/index.cjs`, `test/index.mjs`), `attw --pack .` (`npm run build:test`), or child-process CLI execution when export maps or CLI flags change. |

### Resource Cleanup Checklist for Integration Tests
1. **Ephemeral Ports Only:** Always use `await fastify.listen({ port: 0 })` and read `fastify.listeningOrigin` or `fastify.server.address().port`.
2. **Guaranteed Teardown:** Register `t.after(async () => { await fastify.close() })` immediately after creating the instance or starting the server so failed assertions never leave dangling sockets or timers that hang the test runner.
3. **Stream & Client Cleanup:** Destroy or close client HTTP agents (`undici` Client/Agent, `http.Agent`, WebSocket clients, EventSource connections) inside `t.after()`.
