---
name: fastify-core
description: Use when maintaining, reviewing pull requests, implementing features, or fixing bugs in the fastify/fastify core repository. Covers Fastify core architecture and code (fastify.js, lib/*, build/* generated config-validator.js and error-serializer.js, symbols, avvio encapsulation, FST_ERR_* and FSTWRN*/FSTDEP* codes), core best practices (V8 monomorphic Request/Reply shapes, hot-path lifecycle, fastify.d.ts and types/*.d.ts generics, docs/Reference and Style-Guide.md), core testing (borp 100% coverage, test/types/*.tst.ts with tstyche, webpack/esbuild bundler tests, integration/), and fastify/fastify GitHub Actions workflows (ci.yml, coverage-nix/win, citgm.yml, test-compare, md-lint, links-check, backport). Do not use for standalone plugin repos (use fastify-plugins) or the fastify/workflows repo (use fastify-workflows).
---

# Fastify Core Maintainer Skill (`fastify/fastify`)

Use this skill when reviewing a pull request, implementing a feature, or fixing a bug inside the **[`fastify/fastify`](https://github.com/fastify/fastify)** core framework repository.

> **Companion Skills Boundary:**
> - When reviewing a pull request or diff, pair this skill with [`fastify-code-review`](../fastify-code-review/SKILL.md) for the standardized review flow, severity classification, and Fastify maintainer tone.
> - For `@fastify/*` plugin and utility repositories, use [`fastify-plugins`](../fastify-plugins/SKILL.md).
> - For the centralized reusable workflows repository (`fastify/workflows`), use [`fastify-workflows`](../fastify-workflows/SKILL.md).
> - For the official Fastify website repository (`fastify/website`, `fastify.dev`), use [`fastify-website`](../fastify-website/SKILL.md).

---

## Step 1: Verify Target Branch, Governance & Local Scripts

1. **Branch & Version Line Check:**
   - Inspect `git branch --show-current` and `package.json#version` (`main` / `next` vs maintenance branches `5.x`, `4.x`, etc.).
   - Never bump `package.json#version` in a pull request (`CONTRIBUTING.md`: declaring formal releases is restricted to lead maintainers).
   - **Human-Opened PR Policy (`CONTRIBUTING.md`):** Pull requests in `fastify/fastify` **must** be opened by a human. Prepare verified local patches, conventional PR titles (`pull-request-title.yml`), and review summaries for the human maintainer.
2. **Core Command Reference (`package.json#scripts`):**
   - **Unit tests:** `npm run unit` (`borp`)
   - **Single unit test file:** `npx borp -p test/<file>.test.js`
   - **100% coverage gate (CI):** `npm run coverage:ci-check-coverage` (`borp --coverage --check-coverage --lines 100`)
   - **Type tests:** `npm run test:types` (`tstyche` on `test/types/*.tst.ts`)
   - **Single type test file:** `npx tstyche test/types/<file>.tst.ts`
   - **JS/TS Linter:** `npm run lint` (`eslint` with `neostandard`, `max-len: 120`, `comma-dangle: never`)
   - **Markdown Linter:** `npm run lint:markdown` (`markdownlint-cli2`)
   - **Regenerate Validators & Serializers:** `npm run build:validation`
   - **Verify Generated File Integrity:** `npm run test:validator:integrity`

---

## `fastify/fastify` Repository Layout

```text
.
├── .borp.yaml                      # Test glob: test/**/*.test.js and test/**/*.test.mjs
├── .markdownlint-cli2.yaml         # Markdown linter configuration
├── .npmrc                          # ignore-scripts=true, min-release-age=7, package-lock=false
├── CONTRIBUTING.md                 # Core governance, human-opened PR rule, LTS branches
├── eslint.config.js                # neostandard + max-len 120 + .d.ts & test/types rules
├── fastify.js                      # Main entry point, server factory, routing & exports
├── fastify.d.ts                    # Root TypeScript declarations & server overloads
├── build/
│   ├── build-error-serializer.js   # Generates lib/error-serializer.js via fast-json-stringify
│   ├── build-validation.js         # Generates lib/config-validator.js via @fastify/ajv-compiler
│   └── sync-version.js             # Syncs package.json version into fastify.js VERSION
├── lib/                            # Core runtime modules (context, hooks, reply, request, route, etc.)
├── types/                          # Modular .d.ts files imported by fastify.d.ts
├── test/
│   ├── *.test.js                   # Core unit tests (borp / node:test)
│   ├── same-shape.test.js          # V8 hidden-class / monomorphic shape assertions
│   ├── bundler/                    # webpack and esbuild bundle smoke tests
│   ├── diagnostics-channel/        # node:diagnostics_channel tests
│   ├── http2/ & https/             # HTTP/2 and TLS server tests
│   ├── internals/                  # Direct unit tests of lib/* internals
│   ├── logger/ & stream/           # LoggerController/Pino and payload stream tests
│   └── types/                      # TSTyche type tests (*.tst.ts) + test/types/tsconfig.json
├── integration/                    # Production install + live server smoke test (server.js, test.sh)
├── scripts/
│   └── validate-ecosystem-links.js # Validates GitHub links in docs/Guides/Ecosystem.md
├── docs/
│   ├── Guides/                     # User guides (Style-Guide.md, Ecosystem.md, Contributing.md)
│   └── Reference/                  # API reference (Errors.md, Warnings.md, Hooks.md, Server.md, etc.)
└── .github/workflows/              # 20 core CI, coverage, CITGM, docs, and automation workflows
```

---

## Step 2: Load Core Reference Guides

Load the reference file(s) matching the area of `fastify/fastify` affected by your task:

| Core Area | What It Covers | Reference |
| --- | --- | --- |
| **1. Code & Architecture (`code`)** | `fastify.js` and `lib/*.js` module map, `build/` code generators (`lib/config-validator.js` & `lib/error-serializer.js`) and `test:validator:integrity`, `lib/symbols.js`, `avvio` plugin encapsulation (`lib/plugin-override.js`), `lib/errors.js` (`FST_ERR_*`), and `lib/warnings.js` (`FSTWRN*`, `FSTSEC*`, `FSTDEP*`). | [references/code.md](references/code.md) |
| **2. Best Practices, Performance & Types (`best-practices`)** | V8 monomorphic shapes on `Request`/`Reply` (`test/same-shape.test.js`), request lifecycle hot-path discipline, ESLint (`max-len: 120`, `comma-dangle: never`), `fastify.d.ts` + `types/*.d.ts` 5-parameter generic propagation, and `docs/` synchronizing rules (`Errors.md`, `Warnings.md`, `Style-Guide.md`, `Ecosystem.md`). | [references/best-practices.md](references/best-practices.md) |
| **3. Testing & Type Testing (`test`)** | `borp` unit testing (`npm run unit`) and 100% coverage (`npm run coverage:ci-check-coverage`), `@matteo.collina/tspl` (`await p.completed`), `fastify.inject()` vs `fastify.listen({ port: 0 })`, `test/internals/`, `test/bundler/` (`webpack` & `esbuild`), `integration/`, and `test/types/*.tst.ts` (`tstyche`). | [references/test.md](references/test.md) |
| **4. Workflows & CI Gates (`workflows`)** | All 20 workflows in `fastify/fastify/.github/workflows/`: `ci.yml` (`coverage-nix`, `coverage-win`, `test-unit`, `test-types`, `test-pino-compat`, `package`), `citgm.yml` (`citgm-core-plugins` label), `test-compare.yml`, `missing_types.yml`, `md-lint.yml`, `links-check.yml`, `lint-ecosystem-order.yml`, `pull-request-title.yml`, and `backport.yml`. | [references/workflows.md](references/workflows.md) |

---

## Core Non-Negotiable Checklist Before Completing Any Task

1. **Generated Files Integrity:** If you touch `build/build-validation.js` or `build/build-error-serializer.js`, run `npm run build:validation` and `npm run test:validator:integrity`. Never edit `lib/config-validator.js` or `lib/error-serializer.js` by hand.
2. **Runtime + `types/*.d.ts` + `test/types/*.tst.ts` + `docs/Reference/*.md` Lockstep:** Any change to server options, route options, hooks, `Request`, `Reply`, or `FST_ERR_*` codes must update `fastify.d.ts` / `types/*.d.ts`, `test/types/*.tst.ts`, and `docs/Reference/` in the same patch.
3. **100% Coverage & Linter Cleanliness:** Run `npx borp -p <test-file>`, `npx tstyche <type-test-file>`, `npm run lint`, and `npm run lint:markdown` (when `.md` files change).
