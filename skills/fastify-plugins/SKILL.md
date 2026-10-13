---
name: fastify-plugins
description: Use when creating, structuring, maintaining, reviewing pull requests, implementing features, fixing bugs, or configuring workflows and releases in Fastify plugin repositories (@fastify/* ecosystem plugins and fastify/skeleton-based plugin or utility repos). Covers how Fastify plugin repositories are made and structured, code best practices (fastify-plugin, CJS/ESM export triplet, @fastify/error, decorators, hooks, neostandard, .npmrc), unit and integration tests (node:test, borp, c8 --100, fastify.inject), TypeScript declarations and TSTyche type tests (types/index.d.ts, types/*.tst.ts), and plugin CI workflow and release standards. Do not use for maintaining fastify/fastify core (use fastify-core), fastify/workflows (use fastify-workflows), or fastify/website (use fastify-website).
---

# Fastify Plugins Skill

Use this skill when creating, structuring, reviewing, or maintaining a **Fastify plugin repository** (`@fastify/*` ecosystem plugins or [`fastify/skeleton`](https://github.com/fastify/skeleton)-based plugins and utilities).

> **Companion Skills Boundary:**
> - When reviewing a pull request or diff, pair this skill with [`fastify-code-review`](../fastify-code-review/SKILL.md) for the standardized review flow, severity classification, and Fastify maintainer tone.
> - For the [`fastify/fastify`](https://github.com/fastify/fastify) core framework repository, use [`fastify-core`](../fastify-core/SKILL.md).
> - For the centralized reusable workflows repository ([`fastify/workflows`](https://github.com/fastify/workflows)), use [`fastify-workflows`](../fastify-workflows/SKILL.md).
> - For the official Fastify website repository ([`fastify/website`](https://github.com/fastify/website)), use [`fastify-website`](../fastify-website/SKILL.md).

## Step 1: Inspect Local Plugin Configuration

1. **Inspect Repository Files & Scripts:**
   - Read `package.json`, `.npmrc`, `eslint.config.js` (or `.cjs`), `types/index.d.ts`, `types/*.tst.ts`, and `.github/workflows/*.yml`.
   - Record the exact script names before running commands:
     - **Linter:** `npm run lint` (`eslint` with `neostandard`)
     - **Runtime tests:** `npm run test:unit` (or `npm run unit` in `@fastify/redis`, `npm run test:node` in TypeBox/JSON-Schema type providers)
     - **Type tests:** `npm run test:typescript` (or `npm run typescript` / `npm run test:types`)

---

## How Fastify Plugin Repositories Are Made & Structured

All standardized `@fastify/*` plugin repositories are built from the [`fastify/skeleton`](https://github.com/fastify/skeleton) blueprint:

```text
.
├── .editorconfig                   # 2-space indent, LF line endings, UTF-8
├── .gitattributes                  # text=auto
├── .gitignore                      # Ignores node_modules/, coverage/, .nyc_output/
├── .npmrc                          # ignore-scripts=true, min-release-age=7, package-lock=false
├── .borp.yaml                      # Optional: when using borp instead of c8 + node --test
├── .github/
│   ├── dependabot.yml              # Weekly npm & github-actions updates with 7-day cooldown
│   └── workflows/
│       ├── ci.yml                  # Caller workflow -> fastify/workflows plugins-ci*.yml@<sha>
│       ├── lock-threads.yml        # Caller workflow -> fastify/workflows lock-threads.yml@<sha>
│       ├── package-manager-ci.yml  # Optional caller -> plugins-ci-package-manager.yml@<sha>
│       └── release.yml             # Caller workflow -> fastify/workflows reusable-release.yml@<sha>
├── docs/                           # Optional extended documentation
├── examples/                       # Runnable usage examples
├── eslint.config.js                # neostandard({ ignores: resolveIgnoresFromGitignore(), ts: true })
├── index.js                        # Main plugin entry wrapped with fastify-plugin (fp)
├── lib/                            # Internal helpers, symbols, errors, or sub-modules
├── package.json                    # CommonJS default, types: "types/index.d.ts", 100% test coverage
├── LICENSE                         # MIT License
├── README.md                       # Install, usage, options table, acknowledgments, MIT license
├── test/
│   └── *.test.js                   # node:test suites (100% coverage via c8 --100 or borp -C)
└── types/
    ├── index.d.ts                  # Manual TS declarations + declare module 'fastify' augmentation
    └── index.tst.ts                # TSTyche type tests (imports, options, decorators, negative tests)
```

---

## Step 2: Load Plugin Area References

Load the reference file(s) matching the parts of the plugin repository affected by your task:

| Plugin Area | What It Covers | Reference |
| --- | --- | --- |
| **1. Structure & Code (`code`)** | How a plugin is structured and coded: `package.json` & `.npmrc` baseline, `neostandard` flat ESLint config, `fastify-plugin` (`fp`) metadata, CJS/ESM multi-export triplet (`default` + named export), symbols, registration-time setup, decorators, hooks, `onClose` cleanup, `@fastify/error` codes, and dynamic `import()` caching for ESM dependencies. | [references/code.md](references/code.md) |
| **2. Runtime & Integration Tests (`test`)** | `c8 --100 node --test` and `borp -C --check-coverage` (100% coverage rule), `t.plan(N)` + `t.assert.*`, `@matteo.collina/tspl` (`await p.completed`), `t.after(() => fastify.close())`, `fastify.inject()` vs `fastify.listen({ port: 0 })`, and Docker service tests (`redis`, `postgres`, `mysql`, `mongodb`, `kafka`, `elasticsearch`). | [references/test.md](references/test.md) |
| **3. TypeScript & Type Tests (`type-test`)** | `types/index.d.ts` authoring (`FastifyPluginCallback` / `FastifyPluginAsync`, `declare module 'fastify'`, `declare namespace`, `export =`), dual CJS/ESM TypeScript plugins (`attw --pack .`), and **TSTyche** (`types/index.tst.ts` import matrix, positive/negative `expect().type.*` matchers, and `tstyche/tstyche-skills` integration). | [references/type-test.md](references/type-test.md) |
| **4. Workflow Standards & Releases (`actions`)** | Standard `.github/workflows/` setup in plugin repos (`ci.yml`, `lock-threads.yml`, `package-manager-ci.yml`, `.github/dependabot.yml`), **plugin release configuration & procedure** (`release.yml` calling `reusable-release.yml`, npm OIDC trusted publishing, `release` GitHub Environment reviewer protection, `release:build` hook), and 40-char SHA pinning. | [references/actions.md](references/actions.md) |

---

## Core Best-Practice Checklist for Every Plugin PR / Patch

1. **Code + Types + Tests + Docs Lockstep:** Any new or modified plugin option, decorator, hook, or error code in `index.js` / `lib/` **must** update `types/index.d.ts`, `types/index.tst.ts`, `test/*.test.js`, and `README.md` in the same change.
2. **100% Coverage Discipline:** Almost all Fastify plugins enforce 100% line/branch coverage (`c8 --100` or `borp -C --check-coverage --lines 100`). Cover both happy paths and validation/error branches.
3. **Export Triplet Consistency:** Ensure `index.js` exports `module.exports = plugin`, `module.exports.default = plugin`, and `module.exports.fastify<Name> = plugin`, and verify all import styles in `types/index.tst.ts`.
4. **Standard Caller Workflows & OIDC Releases:** Keep plugin `.github/workflows/` as thin callers of `fastify/workflows` pinned to 40-character commit SHAs, and use `reusable-release.yml` with npm trusted publishing and a protected `release` environment.
