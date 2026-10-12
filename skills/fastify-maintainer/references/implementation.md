# Feature & Bug-Fix Implementation Workflow

Use this workflow when implementing a new feature, fixing a bug, or updating types, tests, CI, or documentation in a Fastify organization repository.

## 1. Discover Conventions & Verify Scope

1. **Complete Repository Discovery:** Follow [repository-discovery.md](repository-discovery.md).
   - If the target repository is a plugin not listed on [https://fastify.dev/ecosystem/](https://fastify.dev/ecosystem/) (unmanned) or has had **no commits in the last 2 years**, stop and warn the maintainer before editing files.
2. **Determine Applicable Areas & Load References:**
   - Load **only** the references needed for the task:
     - Runtime JS & Fastify architecture → [codebase-javascript.md](codebase-javascript.md)
     - `.d.ts` declarations, `.ts` source, or type tests → [typescript.md](typescript.md)
     - Unit, regression, or integration tests → [testing.md](testing.md)
     - CI workflows, docs, or contribution policy → [ci-docs.md](ci-docs.md)

## 2. Step-by-Step Implementation Sequence

### Step A: Reproduce First (For Bug Fixes)
- Write or update a targeted runtime test (or type test for type-only bugs) using the repository's discovered runner (`borp`, `node --test`, `tap`, `tape`, or `tstyche` / `tsd`).
- Run the single test file first and confirm it **fails for the expected reason** before applying the fix.
- If a pre-fix reproduction cannot be executed locally (for example, missing an external database container), document the exact reason.

### Step B: Implement Scoped Runtime Changes
- Follow the local module format (`CommonJS` vs `ESM`) and code style (`neostandard` / `@fastify/eslint-config`).
- Respect Fastify encapsulation, hook/plugin completion rules (never mix `async` and `done()`), `@fastify/error` (`FST_ERR_*`) codes, and hot-path performance (prefer registration-time setup over per-request allocations).
- If the repository generates committed files (such as `npm run build:validation` in `fastify/fastify` or `npm run build` in dual CJS/ESM repos), run the generator script and verify integrity.

### Step C: Synchronize TypeScript Declarations & Type Tests
- Whenever a public option, decorator, hook, method signature, or generic contract changes in a JavaScript repository with `.d.ts` files (`index.d.ts`, `types/index.d.ts`, `fastify.d.ts`), update the `.d.ts` declarations in the same change.
- Add or update type tests using the repository's configured tool (**TSTyche** or `tsd` / `tsc`), covering both positive inference and invalid-usage rejection where appropriate.

### Step D: Choose the Right Test & Integration Boundary
- Default to in-process `fastify.inject()` (`light-my-request`) for routes, hooks, decorators, headers, cookies, and serialization.
- Add a real socket/network integration test (`await fastify.listen({ port: 0 })` with `t.after(() => fastify.close())`) or external service test **only** when the change affects raw socket handling, stream aborts, multipart boundary streaming, HTTP keep-alive/pipelining, or external client drivers.

### Step E: Update Documentation
- If public options, defaults, error codes (`FST_ERR_*`), deprecation codes (`FSTDEP*`), or examples changed, update `README.md` and/or `docs/**/*.md` to match the runtime and TypeScript contracts.

### Step F: Run Repository Validation
Execute the target repository's discovered scripts in order:
1. Targeted runtime test file(s) and targeted type test file(s) (`npx tstyche <path>`).
2. Linter (`npm run lint` — do not auto-format unrelated files).
3. Full unit test suite (`npm run unit` or `npm run test:unit`).
4. Full type test suite (`npm run test:types` or `npm run test:typescript`).
5. Any integrity or package-export checks (`npm run test:validator:integrity`, `npm run build:test` / `attw`).

## 3. Required Implementation Output Structure

Conclude every implementation response with:

1. **Behavior & Compatibility Summary:**
   - What changed, why, and whether the change is semver-patch, semver-minor, or breaking on the target branch.
2. **8-Area Coverage Matrix:**
   - Status of `Codebase`, `JavaScript`, `TypeScript`, `Runtime tests`, `Type tests`, `Integration tests` (with explicit required/not-required rationale), `CI`, and `Docs`.
3. **Validation Evidence:**
   - Pre-fix reproduction result (for bug fixes), exact commands executed, and their outcomes.
   - Honest disclosure of any checks that could not run locally (e.g., unavailable external service containers or OS/Node CI matrix jobs).
4. **Maintainer Patch & PR Readiness (`CON-004`):**
   - Note any repository-specific contribution policy (such as `fastify/fastify`'s requirement that humans open PRs) and provide a concise PR title (matching `action-pr-title` conventions, e.g., `fix: ...` or `feat: ...`) and body summary ready for the maintainer.
