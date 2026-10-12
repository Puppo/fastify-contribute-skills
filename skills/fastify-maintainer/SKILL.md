---
name: fastify-maintainer
description: Use when maintaining repositories in the Fastify GitHub organization—reviewing pull requests, implementing features, or fixing bugs in Fastify core, official fastify.dev/ecosystem plugins, or active Fastify libraries (including repos without fastify in the name such as avvio, middie, under-pressure, point-of-view, light-my-request, fast-json-stringify, fast-uri, otel, session, sse). Do not use for building end-user Fastify applications, publishing releases, or repository administration.
---

# Fastify Maintainer

Use this skill when acting as a maintainer on a repository in the [Fastify GitHub organization](https://github.com/fastify) to **review a pull request / diff** or **implement a feature or bug fix**.

Do **not** activate this skill for general user projects that merely consume Fastify as an application dependency, or for package publishing, release tagging, and repository administration.

## Step 1: Discover Repository Scope, Policies, and Tooling

Always read [references/repository-discovery.md](references/repository-discovery.md) first and verify:

1. **Repository Scope & Activity Guard:**
   - **Official Plugins Rule:** If the repository is a Fastify plugin or type provider (exports a plugin registered via `fastify.register()`, uses `fastify-plugin`, or provides a `FastifyTypeProvider`), verify that it is one of the 60 official Core Plugins listed on [https://fastify.dev/ecosystem/](https://fastify.dev/ecosystem/) (table in `references/repository-discovery.md`). Note that standalone `@fastify/*` utilities (such as `@fastify/error`, `@fastify/busboy`, `@fastify/deepmerge`, `@fastify/send`) are shared libraries, not plugins, and are covered in the non-plugin list in `references/repository-discovery.md`. Any plugin repo in the `fastify` org not on the ecosystem list is **unmanned**—stop and warn the maintainer before proceeding.
   - **2-Year Commit Cutoff:** Run `git log -1 --format="%H %cs"` and `git log -1 --since="2 years ago" --format="%H %cs"` on the default/base branch. If the repository is archived or `--since="2 years ago"` returns no commits, stop and warn the maintainer that the repository is inactive (>2 years without commits).
2. **Local Conventions & Scripts:**
   - Inspect `package.json`, `.github/workflows/`, `CONTRIBUTING.md`, and type-test configuration (`tstyche.config.*`, `tsconfig.json`).
   - Record the exact commands for linting, runtime unit/integration tests, and type tests. Never assume Fastify core script names (`npm run unit` vs `npm run test:unit`, `npm run test:types` vs `npm run test:typescript`).

## Step 2: Select the Workflow

Choose the workflow matching the maintainer's request and load only that workflow file:

- **PR / Diff Review (Read-Only):** Read [references/review.md](references/review.md) when asked to review a PR, branch, commit range, or local diff. Never modify tracked files during a review-only request.
- **Feature / Bug-Fix Implementation:** Read [references/implementation.md](references/implementation.md) when asked to fix a bug, add a feature, update types/tests/docs, or prepare a maintainer patch.

## Step 3: Assess Area Applicability & Load Conditional References

Evaluate all eight maintainer areas against the task or diff. Mark non-applicable areas explicitly with a brief reason instead of inventing missing test suites or imposing new tooling. Load the reference files below **only** when the corresponding area is in play:

| Area | When to Load | Reference |
| --- | --- | --- |
| **1. Codebase** & **2. JavaScript** | Load whenever runtime `.js`/`.mjs`/`.cjs` files, module exports, plugin registration, hooks, encapsulation, errors, or performance-sensitive paths are inspected or modified. | [references/codebase-javascript.md](references/codebase-javascript.md) |
| **3. TypeScript** & **5. Type Tests** | Load whenever `.d.ts` declarations, `.ts` source files, public runtime options/APIs, generics, `TypeProvider` contracts, or type tests (`.tst.ts`, `.test-d.ts`, `tstyche`, `tsd`, `tsc`) are inspected or modified. Also delegate to installed [TSTyche Agent Skills](https://tstyche.org/guides/agent-skills) (`tstyche-type-tests`, `tstyche-project-setup`, `tstyche-migrate-from-tsd`) when present. | [references/typescript.md](references/typescript.md) |
| **4. Runtime Tests** & **6. Integration Tests** | Load whenever reproducing a bug, adding/reviewing unit tests (`borp`, `node --test`, `@matteo.collina/tspl`, `tap`, `tape`), or deciding whether in-process `fastify.inject()` suffices versus real sockets (`listen({ port: 0 })`) or external services. | [references/testing.md](references/testing.md) |
| **7. CI** & **8. Docs** | Load whenever `.github/workflows/`, `package.json` scripts/engines/dependencies, `README.md`, `docs/`, error/deprecation documentation, or contribution/PR policies are affected. | [references/ci-docs.md](references/ci-docs.md) |

## Core Maintainer Principles

1. **Target Checkout is Authoritative:** Follow the checked-out branch's supported Node.js/TypeScript versions, module system, linter (`neostandard` / `@fastify/eslint-config` / `biome`), and test runner.
2. **Runtime + `.d.ts` Alignment:** In Fastify repositories written in JavaScript with manual `.d.ts` files (`index.d.ts`, `types/index.d.ts`, `fastify.d.ts`), any public runtime option, decorator, hook, or return-type change must update `.d.ts` and type tests in lockstep.
3. **Smallest Honest Boundary for Integration Tests:** Default to `fastify.inject()` (`light-my-request`). Require a real listening server (`fastify.listen({ port: 0 })`) + `t.after(() => fastify.close())` or external service containers only when testing socket lifecycle, stream abort, multipart boundaries, HTTP keep-alive/pipelining, or real database/broker drivers.
4. **Honest Verification & Human PR Policy:** Report exact commands run and any checks skipped due to missing local services or matrix environments. Respect Fastify's contribution policy requiring humans to open PRs where applicable—prepare clean patches and PR summaries for the maintainer.
