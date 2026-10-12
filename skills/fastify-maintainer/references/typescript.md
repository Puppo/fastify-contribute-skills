# TypeScript & Type Testing Guidelines

Load this reference when inspecting or modifying `.d.ts` declarations, `.ts` source files, public runtime APIs/options that affect types, Fastify generic/type-provider contracts, or type tests (`tstyche`, `tsd`, `tsc`).

## 1. Runtime JS & `.d.ts` Declaration Synchronization

Most active Fastify repositories (`fastify/fastify`, `@fastify/*` ecosystem plugins, `fastify-plugin`, `light-my-request`, `fast-json-stringify`) author runtime code in CommonJS JavaScript (`.js`) and maintain hand-written TypeScript declarations (`fastify.d.ts`, `index.d.ts`, or `types/index.d.ts`).

1. **Lockstep Updates:**
   - Any change to public plugin options, defaults, decorators (`decorate`, `decorateRequest`, `decorateReply`), hooks, route config, or return types in `.js` **must** update the corresponding `.d.ts` file in the same patch.
   - Check whether option properties are optional (`?`), nullable, or accept union/callback forms at runtime—ensure the `.d.ts` interface matches the runtime validation (`config-validator` / schema / `typeof` checks).
2. **Fastify Plugin Typing & Module Augmentation:**
   - Official `@fastify/*` plugins typically export a `FastifyPluginCallback<Options>` or `FastifyPluginAsync<Options>` (from `'fastify'`) and augment the `'fastify'` module when adding decorators or route options:
     ```ts
     import { FastifyPluginAsync } from 'fastify'

     export namespace myPlugin {
       export interface MyPluginOptions {
         prefix?: string
       }
     }

     declare module 'fastify' {
       interface FastifyInstance {
         myHelper(): string
       }
       interface FastifyRequest {
         myRequestProp: string | null
       }
     }

     export const myPlugin: FastifyPluginAsync<myPlugin.MyPluginOptions>
     export default myPlugin
     ```
   - Ensure both default and named exports in `types/index.d.ts` match what the CommonJS `index.js` actually assigns (`module.exports = fp(...)`, `module.exports.default = ...`, named exports).
3. **Generic Propagation (`RawServer`, `TypeProvider`, `SchemaCompiler`, `Logger`):**
   - In `fastify/fastify` core and type providers (`@fastify/type-provider-typebox`, `@fastify/type-provider-json-schema-to-ts`), preserve generic parameters (`RawServer extends RawServerBase = RawServerDefault`, `TypeProvider extends FastifyTypeProvider = FastifyTypeProviderDefault`, `SchemaCompiler extends FastifySchema = FastifySchema`, `Logger extends FastifyBaseLogger = FastifyBaseLogger`) across overloads and chained methods (`register`, `after`, `route`, `get`, `post`, `withTypeProvider`).
   - In dual CJS/ESM TypeScript packages (such as `fastify-type-provider-typebox`), run `npm run build` and `npm run build:test` (`attw --pack .`) if configured to verify export resolution across `node16`/`nodenext`/`bundler`.

---

## 2. Type Tests (`tstyche`, `tsd`, and `tsc`)

Active Fastify repositories primarily use **[TSTyche](https://tstyche.org/)** (`tstyche` via `npm run test:types` or `npm run test:typescript`) with test files in `test/types/*.test-d.ts` or `types/index.test-d.ts` (or `*.tst.ts`). A few repositories use `tsc --project ./test/types/tsconfig.json` (e.g., `avvio`) or `tsd` on older maintenance branches.

### Companion Skill Integration: [TSTyche Agent Skills](https://tstyche.org/guides/agent-skills)

When working with TSTyche type tests or migrating a Fastify repository from `tsd` to `tstyche`:
- Check if the companion skills from [`tstyche/tstyche-skills`](https://github.com/tstyche/tstyche-skills) ([https://tstyche.org/guides/agent-skills](https://tstyche.org/guides/agent-skills)) are installed in the agent environment:
  - `tstyche-type-tests`: writing, reviewing, and debugging TSTyche assertions, helpers, directives, and inference.
  - `tstyche-project-setup`: configuring `tstyche.config.*`, CLI flags, and CI.
  - `tstyche-migrate-from-tsd`: migrating `.test-d.ts` files and `tsd` assertions to TSTyche without losing test intent.
- If installed, load the relevant `tstyche-*` skill. If not installed, follow the self-contained TSTyche rules below (and recommend `npx skills add tstyche/tstyche-skills` to the maintainer when deep TSTyche work is needed).

### Self-Contained TSTyche Rules (`tstyche`)

1. **Inspect Config First:**
   - Check `package.json#scripts` (`test:types` or `test:typescript`), `tstyche.config.*` (often configured with `testFileMatch: ['test/types/**/*.test-d.ts']` or `['types/**/*.test-d.ts']` in Fastify repos), and the applicable `tsconfig.json`.
2. **Core Assertion Syntax:**
   - Import `expect`, `test`, `describe` from `'tstyche'` (and `import type { _ } from 'tstyche'` when testing generic instantiation):
     ```ts
     import { expect, test } from 'tstyche'
     import fastify from 'fastify'

     test('plugin option and decorator inference', async () => {
       const app = fastify()
       expect(app.myHelper()).type.toBe<string>()
       expect(app.myHelper).type.not.toBeCallableWith('unexpected-arg')
     })
     ```
   - **Relation Matchers:** `.toBe<Target>()`, `.toBeAssignableTo<Target>()`, `.toBeAssignableFrom<Target>()` (each accepts either a type argument or a value expression). Keep the expression/type under test on the left (`expect(actual).type.toBe<Expected>()`).
   - **Ability Matchers (Prefer over `@ts-expect-error` or deprecated `.toRaiseError`):**
     - `.toBeCallableWith(...)` / `.not.toBeCallableWith(...)`
     - `.toBeConstructableWith(...)` / `.not.toBeConstructableWith(...)`
     - `.toBeInstantiableWith<[Arg1, Arg2]>()` / `.not.toBeInstantiableWith<...>()`
     - `.toHaveProperty('propName')` / `.not.toHaveProperty('propName')`
   - **Avoid `.toRaiseError`:** `.toRaiseError` is deprecated in TSTyche; use the specific negative ability matcher (e.g., `.not.toBeCallableWith(...)`) for new assertions, and only preserve `.toRaiseError` when touching legacy lines that are not being migrated.
   - **`any` and `never` Protection:** TSTyche enables `rejectAnyType` and `rejectNeverType` by default so inferred types cannot silently degrade to `any` or `never`.
3. **Positive + Negative Coverage:**
   - Every type change or type bug fix must test **both**:
     1. **Positive cases:** valid options, inferred return types, narrowed route request/reply types.
     2. **Negative cases:** invalid option types, missing required options, or disallowed overload combinations.
   - For type regressions, verify that the new type assertion **fails before the `.d.ts`/`.ts` fix** and **passes after the fix**.
4. **Fast Targeted Execution:**
   - Run the single type-test file first (`npx tstyche path/to/file.test-d.ts`), then run the repository's full type-test script (`npm run test:types` or `npm run test:typescript`).

### Repositories Using `tsd` or `tsc`

- **`tsd` (Legacy / Maintenance Branches):**
  - Uses `import { expectType, expectAssignable, expectNotAssignable, expectError, expectDeprecated } from 'tsd'`.
  - Keep `expectError(...)` as narrow as possible so an outer call does not mask which argument failed.
- **`tsc --project` (e.g., `avvio`):**
  - Type tests are compiled directly via `npm run test:typescript` (`tsc --project ./test/types/tsconfig.json`). Use explicit type annotations and `// @ts-expect-error` for negative cases.
