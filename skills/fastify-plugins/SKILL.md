---
name: fastify-plugins
description: Maintain, implement, or review Fastify ecosystem plugins, type providers, and standalone utilities, including their code, tests, public types, CI, and releases. Distinguishes registered plugins from utility APIs and follows the target repository's configuration. Use fastify-core for fastify/fastify, fastify-workflows for reusable workflow internals, and fastify-website for the website.
---

# Fastify Plugins Skill

Use this skill when creating, structuring, reviewing, or maintaining **Fastify ecosystem plugins, type providers, or standalone utilities**, including `fastify-plugin`, `@fastify/error`, and `@fastify/deepmerge`.

> **Companion Skills Boundary:**
> - When reviewing a pull request or diff, pair this skill with [`fastify-code-review`](../fastify-code-review/SKILL.md) for the standardized review flow, severity classification, and Fastify maintainer tone.
> - For the [`fastify/fastify`](https://github.com/fastify/fastify) core framework repository, use [`fastify-core`](../fastify-core/SKILL.md).
> - For the centralized reusable workflows repository ([`fastify/workflows`](https://github.com/fastify/workflows)), use [`fastify-workflows`](../fastify-workflows/SKILL.md).
> - For the official Fastify website repository ([`fastify/website`](https://github.com/fastify/website)), use [`fastify-website`](../fastify-website/SKILL.md).

Repository maps, dependency versions, and workflow defaults below are examples; confirm them in the target checkout. Blueprint source root observed on 2026-10-14: [fastify/skeleton at `faa3f75`](https://github.com/fastify/skeleton/tree/faa3f75d001739c9f1fae00dd20aab1175341898). Official documentation for the supported Fastify version and package tests determine behavior.

## Step 1: Classify the Package & Inspect Local Configuration

1. **Inspect Repository Files & Scripts:**
   - Read contribution rules, `package.json` entrypoints/exports/scripts, `.npmrc`, ignore rules, linter configuration, declarations, tests, and workflows in the checkout. Local configuration takes precedence over the examples below.
   - Record the exact script names before running commands:
     - **Linter:** `npm run lint` (`eslint` with `neostandard`)
     - **Runtime tests:** `npm run test:unit` (or `npm run unit` in `@fastify/redis`, `npm run test:node` in TypeBox/JSON-Schema type providers)
     - **Type tests:** `npm run test:typescript` (or `npm run typescript` / `npm run test:types`)
2. **Classify by Actual API:**
   - **Registered plugins:** Apply wrapping, metadata, hooks, decorators, and registration tests only where the plugin's scope and behavior require them. Preserve intentional encapsulation.
   - **Standalone utilities:** Preserve existing functions/classes, module exports, declaration layout, test runner, and release process. Do not add `fastify-plugin`, Fastify augmentations, or an export triplet merely to match the plugin template. Test through the utility's own API.
   - **Type providers / TypeScript packages:** Follow their source/build/exports configuration and generated declarations; do not replace them with a hand-written CommonJS plugin layout.

---

## How Fastify Plugin Repositories Are Made & Structured

The [`fastify/skeleton`](https://github.com/fastify/skeleton) blueprint illustrates a common plugin layout. Use it for applicable new plugins; preserve an existing package's supported structure and tooling:

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

## Best-Practice Checklist for the Affected Package

1. **Code, Types, Tests & Docs:** Test changed runtime behavior and regressions. When public contracts change, update the relevant declarations, type tests, and documentation using existing paths. An internal fix need not change an unchanged public declaration.
2. **Coverage Discipline:** Preserve metrics and thresholds from scripts, runner configuration, and CI. Distinguish line-only gates from branch/function gates; report coverage separately from ordinary test results.
3. **Export Compatibility:** Preserve supported imports. Check the CJS/default/named triplet where a plugin exposes it, and the actual exports for utilities and type providers.
4. **Workflow & Release Compatibility:** Follow the checkout's caller workflows and release process. For repositories using `fastify/workflows`, keep external references SHA-pinned and preserve their trusted-publishing and release-environment setup.
