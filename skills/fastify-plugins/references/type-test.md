# Fastify Plugin TypeScript Declarations & TSTyche Type Tests (`type-test`)

Load this reference when authoring, updating, or reviewing `types/index.d.ts`, `types/*.tst.ts` (or `*.test-d.ts`), or TypeScript sources in a Fastify plugin repository.

---

## 1. Canonical `types/index.d.ts` Pattern for Fastify Plugins

Almost all `@fastify/*` plugins are written in CommonJS JavaScript (`index.js`) with hand-maintained TypeScript declarations in `types/index.d.ts` (`"types": "types/index.d.ts"` in `package.json`).

### Standard `types/index.d.ts` Template
Every plugin `types/index.d.ts` must:
1. Import `FastifyPluginCallback` (for callback plugins) or `FastifyPluginAsync` (for async plugins) from `'fastify'`.
2. Use `declare module 'fastify'` to augment `FastifyInstance`, `FastifyRequest`, `FastifyReply`, or `FastifyContextConfig` with any decorators or route config added by the plugin.
3. Declare a namespace (`declare namespace fastifyMyPlugin`) exporting the options interface, any helper types/constants, the named export `export const fastifyMyPlugin: FastifyMyPlugin`, and `export { fastifyMyPlugin as default }`.
4. Declare the callable function and `export = fastifyMyPlugin` at the bottom so CommonJS `require`, `import = require()`, ESM default import, ESM named import, and namespace `import * as` all type-check simultaneously:

```ts
import {
  FastifyPluginCallback,
  FastifyPluginAsync,
  FastifyRequest,
  FastifyReply
} from 'fastify'

type FastifyMyPlugin = FastifyPluginCallback<
  NonNullable<fastifyMyPlugin.FastifyMyPluginOptions>
>
// Or for async plugins:
// type FastifyMyPlugin = FastifyPluginAsync<NonNullable<fastifyMyPlugin.FastifyMyPluginOptions>>

declare module 'fastify' {
  interface FastifyInstance {
    myPluginHelper(input: string): string;
  }

  interface FastifyRequest {
    myPluginData: string | null;
  }

  interface FastifyReply {
    sendMyPluginResponse(payload: string): FastifyReply;
  }
}

declare namespace fastifyMyPlugin {
  export interface FastifyMyPluginOptions {
    /**
     * Custom option description.
     * @default 'defaultValue'
     */
    customOption?: string;
    hook?: 'onRequest' | 'preParsing' | 'preValidation' | 'preHandler' | false;
  }

  export const fastifyMyPlugin: FastifyMyPlugin
  export { fastifyMyPlugin as default }
}

declare function fastifyMyPlugin (
  ...params: Parameters<FastifyMyPlugin>
): ReturnType<FastifyMyPlugin>

export = fastifyMyPlugin
```

### Synchronizing Runtime JS and `types/index.d.ts`
- **Options Parity:** Every option read from `options.*` in `index.js` / `lib/` must exist in `Fastify<Plugin>Options` with the exact union of accepted runtime types (e.g., `boolean | 'auto'`, `string | string[] | Buffer | Buffer[]`, `false` when disabling a hook).
- **Named & Secondary Exports Parity:** If `index.js` attaches extra properties to `module.exports` (such as `module.exports.ajvFilePlugin = ajvFilePlugin` in `@fastify/multipart`, `module.exports.httpErrors = httpErrors` in `@fastify/sensible`, or `module.exports.Signer = Signer` in `@fastify/cookie`), export them inside `declare namespace fastify<Plugin>` (and intersect the plugin type if accessed as properties on the default export).
- **JSDoc in `.js` Entry Point:** Optionally annotate the runtime plugin function in `index.js` with `/** @type {typeof import('./types/index').fastifyMyPlugin} */` (as in `@fastify/sensible`).

---

## 2. TypeScript-Authored Dual CJS/ESM Plugins (Type Providers)

For TypeScript-authored plugins/type providers like `@fastify/type-provider-typebox` and `@fastify/type-provider-json-schema-to-ts`:
- Source lives in `index.ts` and compiles to `dist/cjs/` (`CommonJS`) and `dist/esm/` (`NodeNext`) via `npm run build` (`tsc` + `post-build.js`).
- `package.json#exports` defines separate `"import"` (`./dist/esm/index.d.mts` + `./dist/esm/index.mjs`) and `"require"` (`./dist/cjs/index.d.ts` + `./dist/cjs/index.js`) conditions.
- Always run `npm run build` (which runs `attw --pack .` via `@arethetypeswrong/cli`) before running `npm run test:types` (`tstyche`) so `dist/` declarations are up to date and verified across module resolution modes.

---

## 3. Type Testing with TSTyche (`types/*.tst.ts`)

All active Fastify plugins use **[TSTyche](https://tstyche.org/)** (`"test:typescript": "tstyche"` in `package.json`) with type test files placed in `types/index.tst.ts` (and optional focused files like `types/named-import.tst.ts`, `types/imports.tst.ts`, `types/namespace.tst.ts`).

> **Companion Skills ([TSTyche Agent Skills](https://tstyche.org/guides/agent-skills)):**
> Check if `tstyche-type-tests`, `tstyche-project-setup`, or `tstyche-migrate-from-tsd` from [`tstyche/tstyche-skills`](https://github.com/tstyche/tstyche-skills) are installed in the agent environment. If installed, load them for complex TSTyche tasks; otherwise follow the built-in rules below.

### 1. Default TSTyche Discovery in Fastify Plugins
- By default, `tstyche` automatically discovers `types/**/*.tst.ts` (and `types/**/*.tst.mts` or `**/*.test-d.ts`) without requiring a `tstyche.config.json` file!
- Run all type tests:
  ```bash
  npm run test:typescript   # or `npx tstyche`
  ```
- Run a single type test file:
  ```bash
  npx tstyche types/index.tst.ts
  ```

### 2. Testing All Import Styles (The Import Matrix)
At the top of `types/index.tst.ts` (or in `types/imports.tst.ts`), verify that every supported CJS and ESM import style resolves to the plugin type and registers cleanly on a `FastifyInstance` (pattern from `@fastify/cookie`, `@fastify/multipart`, `@fastify/postgres`):

```ts
import { expect } from 'tstyche'
import * as fastifyMyPluginStar from '..'
import fastifyMyPluginCjsImport = require('..')
import myPluginDefault, { fastifyMyPlugin as myPluginNamed } from '..'
import fastify, { type FastifyInstance, type FastifyReply } from 'fastify'

const fastifyMyPluginCjs = require('..')

const app: FastifyInstance = fastify()
app.register(myPluginNamed)
app.register(myPluginDefault)
app.register(fastifyMyPluginCjs)
app.register(fastifyMyPluginCjsImport.default)
app.register(fastifyMyPluginCjsImport.fastifyMyPlugin)
app.register(fastifyMyPluginStar.default)
app.register(fastifyMyPluginStar.fastifyMyPlugin)

expect(myPluginNamed).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
expect(myPluginDefault).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
expect(fastifyMyPluginCjsImport.default).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
expect(fastifyMyPluginCjsImport.fastifyMyPlugin).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
expect(fastifyMyPluginStar.default).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
expect(fastifyMyPluginStar.fastifyMyPlugin).type.toBe<fastifyMyPluginStar.FastifyMyPlugin>()
```

### 3. Writing Positive and Negative TSTyche Assertions
Use `expect(expression).type.*` from `'tstyche'` to test plugin options, augmented decorators on `fastify`/`request`/`reply`, and invalid argument rejection:

```ts
// 1. Positive option & decorator assertions
app.register(myPluginDefault, {
  customOption: 'hello',
  hook: 'onRequest'
})

app.after((_err) => {
  expect(app.myPluginHelper('test')).type.toBe<string>()

  app.get('/', (request, reply) => {
    expect(request.myPluginData).type.toBe<string | null>()
    expect(reply.sendMyPluginResponse('ok')).type.toBe<FastifyReply>()
    expect(reply.sendMyPluginResponse('ok')).type.toBeAssignableTo<FastifyReply>()
  })
})

// 2. Negative assertions (use ability matchers, NOT deprecated .toRaiseError)
expect(app.myPluginHelper).type.not.toBeCallableWith(123)
expect(app.myPluginHelper).type.not.toBeCallableWith()
expect(app.register).type.not.toBeCallableWith(myPluginDefault, {
  customOption: 12345
})
```

### 4. Key TSTyche Matchers Reference
- **Type Identity & Assignability:**
  - `expect(actual).type.toBe<Expected>()`
  - `expect(actual).type.toBeAssignableTo<Expected>()`
  - `expect(actual).type.toBeAssignableFrom<Expected>()`
- **Ability Matchers (Preferred for Negative Tests):**
  - `expect(fn).type.toBeCallableWith(...)` / `expect(fn).type.not.toBeCallableWith(...)`
  - `expect(Cls).type.toBeConstructableWith(...)` / `expect(Cls).type.not.toBeConstructableWith(...)`
  - `expect(obj).type.toHaveProperty('key')` / `expect(obj).type.not.toHaveProperty('key')`
- **Avoid `.toRaiseError()`:** `.toRaiseError()` is deprecated in TSTyche v7; always prefer `.not.toBeCallableWith(...)` or specific negative ability matchers for new type tests.
