# Fastify Plugin Code & Configuration Patterns (`code`)

Load this reference when reviewing or implementing runtime code (`index.js`, `lib/*.js`), `package.json`, `.npmrc`, or `eslint.config.js` in a Fastify plugin repository.

---

## 1. Standard `package.json`, `.npmrc` & Linter Setup

### Canonical `package.json` Structure
Official `@fastify/*` plugins are authored in CommonJS by default (`"type": "commonjs"`), ship hand-written TypeScript declarations at `"types": "types/index.d.ts"`, and configure `eslint` (`neostandard`), `c8`/`borp`, and `tstyche`:

```json
{
  "name": "@fastify/my-plugin",
  "version": "1.0.0",
  "description": "Fastify plugin for ...",
  "main": "index.js",
  "type": "commonjs",
  "types": "types/index.d.ts",
  "scripts": {
    "lint": "eslint",
    "lint:fix": "eslint --fix",
    "test": "npm run test:unit && npm run test:typescript",
    "test:typescript": "tstyche",
    "test:unit": "c8 --100 node --test"
  },
  "repository": {
    "type": "git",
    "url": "git+https://github.com/fastify/fastify-my-plugin.git"
  },
  "keywords": [
    "fastify",
    "plugin"
  ],
  "license": "MIT",
  "funding": [
    {
      "type": "github",
      "url": "https://github.com/sponsors/fastify"
    },
    {
      "type": "opencollective",
      "url": "https://opencollective.com/fastify"
    }
  ],
  "dependencies": {
    "@fastify/error": "^4.2.0",
    "fastify-plugin": "^6.0.0"
  },
  "devDependencies": {
    "@types/node": "^26.0.1",
    "c8": "^12.0.0",
    "eslint": "^9.17.0",
    "fastify": "^5.0.0",
    "neostandard": "^0.13.0",
    "tstyche": "^7.1.0"
  },
  "publishConfig": {
    "access": "public"
  }
}
```

> **Note on `borp` Variant:** Several plugins (`@fastify/sensible`, `@fastify/static`, `@fastify/under-pressure`, `@fastify/autoload`) use `borp` instead of `c8 --100 node --test`:
> `"test:unit": "borp -C --check-coverage --lines 100 --reporter=@jsumners/line-reporter"` with a `.borp.yaml` file containing `files: ['test/**/*.test.js']`. Always preserve the existing runner in `package.json`.

### `.npmrc` Supply-Chain Hardening
Every Fastify plugin repository includes a `.npmrc` file at the repository root with these three settings (from `fastify/skeleton`):

```ini
ignore-scripts=true
min-release-age=7
package-lock=false
```

- Do **not** commit `package-lock.json` in Fastify plugin repositories (`package-lock=false`).

### Flat ESLint Configuration (`eslint.config.js`)
Active Fastify plugins use ESLint v9 flat config powered by `neostandard` with TypeScript linting enabled (`ts: true` so `types/*.d.ts` and `types/*.tst.ts` are linted too):

```js
'use strict'

module.exports = require('neostandard')({
  ignores: require('neostandard').resolveIgnoresFromGitignore(),
  ts: true
})
```

---

## 2. Plugin Entry Point, `fastify-plugin` (`fp`), and Export Triplet

### File Header & Built-in Imports
- Begin every CommonJS `.js` file with `'use strict'`.
- Always use the `node:` protocol prefix when importing Node.js core modules (`require('node:fs')`, `require('node:path')`, `require('node:stream/promises')`, `require('node:crypto')`).

### Wrapping with `fastify-plugin` (`fp`)
Plugins that add decorators (`decorate`, `decorateRequest`, `decorateReply`), hooks, or content-type parsers intended to be visible outside the plugin's own scope **must** wrap the plugin function with `fastify-plugin`:

```js
'use strict'

const fp = require('fastify-plugin')

function fastifyMyPlugin (fastify, options, done) {
  // plugin registration logic
  done()
}

const plugin = fp(fastifyMyPlugin, {
  fastify: '5.x',
  name: '@fastify/my-plugin',
  // Optional: declare required peer plugins and decorators checked by fastify-plugin
  dependencies: ['@fastify/cookie'],
  decorators: {
    fastify: ['someDecorator'],
    request: ['cookies'],
    reply: ['setCookie']
  }
})
```

- **When NOT to wrap with `fp`:** If a plugin registers routes (e.g., `@fastify/swagger-ui`) and relies on route prefixing (`fastify.register(plugin, { prefix: '/docs' })`) or scoped encapsulation, keep the route-registering sub-plugin encapsulated.

### Canonical Multi-Export Triplet
To support CommonJS `require('@fastify/my-plugin')`, destructured `const { fastifyMyPlugin } = require('@fastify/my-plugin')`, ESM default `import fastifyMyPlugin from '@fastify/my-plugin'`, and ESM named `import { fastifyMyPlugin } from '@fastify/my-plugin'`, always export the triplet at the bottom of `index.js`:

```js
/**
 * These export configurations enable JS and TS developers
 * to consume @fastify/my-plugin in whatever way best suits their needs.
 */
module.exports = plugin
module.exports.default = plugin
module.exports.fastifyMyPlugin = plugin

// Attach any extra utilities, constants, or Ajv plugins after the triplet:
module.exports.myHelper = myHelper
```

---

## 3. Plugin Lifecycle, Hooks, Decorators & Resource Management

### 1. Never Mix `async` and `done`
- Choose **either**:
  - Callback signature: `function fastifyMyPlugin (fastify, options, done) { ... done() }`
  - Async signature: `async function fastifyMyPlugin (fastify, options) { ... }`
- **Never** declare an `async` plugin or hook function that also accepts or calls `done()` / `next()`. Calling `done()` inside an `async` function causes `avvio` / Fastify double-callback errors.
- In callback-style functions, every early-exit validation branch must `return done(new FST_ERR_...())` so execution stops immediately.

### 2. Symbol-Keyed Internal State & Safe Request/Reply Decorators
- Define module-scoped `Symbol` keys for internal request/reply state to prevent property collisions:
  ```js
  const kReplyState = Symbol('fastify.myPlugin.replyState')
  ```
- Pre-declare all request and reply decorators at registration time with `null` or primitive defaults:
  ```js
  fastify.decorateRequest('myData', null)
  fastify.decorateReply(kReplyState, null)
  ```
- **Never** pass a mutable object or array directly to `decorateRequest('myData', {})` or `decorateReply('items', [])`—that single instance would be shared across all concurrent requests! Initialize per-request containers inside an `onRequest` or `preParsing` hook instead.

### 3. Registration-Time vs Request-Time Performance
- Do all option parsing, default merging, schema registration (`fastify.addSchema`), regex compilation, and function factory setup **once** inside the plugin registration body (or `onRoute` hook).
- Avoid `Object.assign`, closure allocation, or option validation inside hot per-request hooks (`onRequest`, `preHandler`, `onSend`) unless request-specific options are required.
- Avoid `async` functions on synchronous hot-path hooks (`onRequest`, `onSend`) when synchronous callback hooks (`(req, reply, done) => { ...; done() }`) can run without allocating a `Promise` on every request (see `@fastify/cookie` `onReqHandlerWrapper` and `fastifyCookieOnSendHandler`).

### 4. Loading ESM-Only Dependencies in CommonJS Plugins
When an upstream runtime dependency switches to ESM-only (such as `cookie` in `@fastify/cookie`) while the plugin still supports CommonJS:
- Make the plugin registration function `async` and dynamically `await import(...)` **once** at registration time, caching the module in a module-scoped variable so request-time calls remain synchronous:
  ```js
  let externalEsmModule = null
  async function loadExternalModule () {
    if (externalEsmModule == null) {
      externalEsmModule = await import('some-esm-pkg')
    }
  }

  async function plugin (fastify, options) {
    await loadExternalModule()
    // ...
  }
  ```

### 5. Resource Teardown with `onClose`
- Plugins that create database/cache clients, connection pools, intervals, or background workers (`@fastify/redis`, `@fastify/postgres`, `@fastify/mongodb`, `@fastify/under-pressure`, `@fastify/schedule`) **must** register an `onClose` hook to release resources when `fastify.close()` runs:
  ```js
  fastify.addHook('onClose', (_fastify, done) => {
    clearTimeout(timer)
    client.quit(done)
  })
  ```

---

## 4. Standardized Error Handling (`@fastify/error`)

Use `@fastify/error` instead of generic `new Error()` for plugin configuration and runtime errors:

```js
'use strict'

const createError = require('@fastify/error')

const FST_MY_PLUGIN_INVALID_OPTION = createError(
  'FST_MY_PLUGIN_INVALID_OPTION',
  'Invalid option "%s" provided to @fastify/my-plugin',
  500,
  TypeError
)

const FST_MY_PLUGIN_UNAUTHORIZED = createError(
  'FST_MY_PLUGIN_UNAUTHORIZED',
  'Missing or invalid credentials',
  401
)
```

- Instantiate errors at module top-level (not inside request handlers).
- Export error classes or error maps when consumers or TypeScript declarations (`types/index.d.ts`) reference them (e.g., `@fastify/bearer-auth`, `@fastify/multipart`, `@fastify/jwt`).
