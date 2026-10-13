# Fastify Maintainer Agent Skills (`fastify-contribute-skills`)

Portable [Agent Skills](https://agentskills.io/specification) (`SKILL.md`) designed for [Fastify](https://github.com/fastify) maintainers to **review pull requests**, **implement features or bug fixes**, and **maintain workflows and documentation** across the Fastify ecosystem.

## Available Skills

### 1. [`fastify-code-review`](skills/fastify-code-review/SKILL.md) — Standardized PR Code Review Across All Fastify Repos
Dedicated skill that standardizes **how** pull requests are reviewed across every Fastify repository, ensuring a consistent read-only flow, technical checklist verification, and the welcoming, concise Fastify maintainer tone:
- **6-Step Review Flow & Checklists ([`references/review-flow-and-checklists.md`](skills/fastify-code-review/references/review-flow-and-checklists.md))**: Read-only diff/PR inspection (`gh pr view`, `gh pr diff`, `git diff`), organization-wide governance checks (`fastify/action-pr-title` Conventional Commits, no `package.json#version` bump in PRs, `.npmrc` lockfile & generated artifact hygiene, semver/branch checks), and repo-specific checklists paired with `fastify-core`, `fastify-plugins`, `fastify-workflows`, or `fastify-website`.
- **Maintainer Tone & Output Templates ([`references/tone-and-templates.md`](skills/fastify-code-review/references/tone-and-templates.md))**: Fastify maintainer voice rules (welcoming, concise, collaborative questions, zero linter duplication, explicit `LGTM` on clean PRs), severity tags (`[Blocking]`, `[Suggestion]`, `[Question]`), GitHub ````suggestion` blocks, the 4-part review output template, and canonical maintainer comment examples.

### 2. [`fastify-core`](skills/fastify-core/SKILL.md) — `fastify/fastify` Core Repository
Dedicated skill for maintaining, reviewing PRs, implementing features, and fixing bugs in the [`fastify/fastify`](https://github.com/fastify/fastify) core framework repository:
- **Code & Subsystem Architecture ([`references/code.md`](skills/fastify-core/references/code.md))**: `fastify.js` and `lib/*.js` module map, `build/` standalone code generators (`lib/config-validator.js` & `lib/error-serializer.js`) and `npm run test:validator:integrity`, `lib/symbols.js`, `avvio` plugin override (`lib/plugin-override.js`), `lib/errors.js` (`FST_ERR_*`), and `lib/warnings.js` (`FSTWRN*`, `FSTSEC*`, `FSTDEP*`).
- **Best Practices, Performance & Types ([`references/best-practices.md`](skills/fastify-core/references/best-practices.md))**: V8 monomorphic hidden classes on `Request`/`Reply` (`test/same-shape.test.js`), request lifecycle hot-path discipline, ESLint (`neostandard({ ts: true })`, `max-len: 120`, `comma-dangle: never`), `fastify.d.ts` + modular `types/*.d.ts` generic propagation, and `docs/` synchronization (`Errors.md`, `Warnings.md`, `Style-Guide.md`, `Ecosystem.md`).
- **Testing & Type Testing ([`references/test.md`](skills/fastify-core/references/test.md))**: `borp` unit testing (`npm run unit`) and 100% line coverage (`npm run coverage:ci-check-coverage`), `@matteo.collina/tspl` (`await p.completed`), `fastify.inject()` vs `fastify.listen({ port: 0 })`, `test/internals/`, `test/bundler/` (`webpack` & `esbuild`), `integration/`, and `test/types/*.tst.ts` (`tstyche`).
- **Workflows & CI Gates ([`references/workflows.md`](skills/fastify-core/references/workflows.md))**: All 20 workflows in `fastify/fastify/.github/workflows/` (`ci.yml`, `citgm.yml`, `test-compare.yml`, `missing_types.yml`, `md-lint.yml`, `links-check.yml`, `lint-ecosystem-order.yml`, `validate-ecosystem-links.yml`, `pull-request-title.yml`, `notify-website-update.yml`, and `backport.yml`).

### 3. [`fastify-plugins`](skills/fastify-plugins/SKILL.md) — `@fastify/*` Plugin & Utility Repositories
Dedicated skill for creating, structuring, reviewing, and maintaining `@fastify/*` ecosystem plugins and [`fastify/skeleton`](https://github.com/fastify/skeleton)-based repositories:
- **Structure & Code ([`references/code.md`](skills/fastify-plugins/references/code.md))**: `fastify/skeleton` layout, `package.json` & `.npmrc` (`ignore-scripts=true`, `min-release-age=7`, `package-lock=false`), `neostandard` flat ESLint config (`ts: true`), `fastify-plugin` (`fp`) metadata, CJS/ESM export triplet (`module.exports`, `.default`, `.fastify<Name>`), symbol-keyed state, registration-time setup, `onClose` teardown, `@fastify/error`, and dynamic `import()` caching for ESM dependencies.
- **Tests ([`references/test.md`](skills/fastify-plugins/references/test.md))**: `c8 --100 node --test` and `borp -C --check-coverage` (100% coverage rule), `t.plan(N)` + `t.assert.*`, `@matteo.collina/tspl`, `t.after(() => fastify.close())`, `fastify.inject()` vs ephemeral `fastify.listen({ port: 0 })`, and Docker service container tests.
- **Type Tests ([`references/type-test.md`](skills/fastify-plugins/references/type-test.md))**: `types/index.d.ts` authoring (`FastifyPluginCallback` / `FastifyPluginAsync`, `declare module 'fastify'`, `declare namespace`, `export =`), dual CJS/ESM TypeScript plugins (`attw --pack .`), and **TSTyche** (`types/index.tst.ts` 7-import matrix and positive/negative `expect().type.*` assertions).
- **Workflow Standards & Releases ([`references/actions.md`](skills/fastify-plugins/references/actions.md))**: Caller workflows in `.github/workflows/` (`ci.yml`, `lock-threads.yml`, `package-manager-ci.yml`), `.github/dependabot.yml`, and plugin releases (`release.yml` calling `reusable-release.yml`, npm OIDC trusted publishing, `release` GitHub Environment reviewer protection, and `release:build` hooks).

### 4. [`fastify-workflows`](skills/fastify-workflows/SKILL.md) — `fastify/workflows` Repository
Dedicated skill for maintaining the centralized [`fastify/workflows`](https://github.com/fastify/workflows) (`@fastify/workflows`) repository itself:
- **Reusable CI & Service Workflows ([`references/ci-workflows.md`](skills/fastify-workflows/references/ci-workflows.md))**: `plugins-ci.yml`, `nested-quality.yml`, `plugins-ci-package-manager.yml`, and all 6 service-container workflows (`redis`, `postgres`, `mysql`, `mongo`, `kafka`, `elasticsearch`)—covering shared input contracts, Node/OS/service matrices, healthchecks, and Dependabot `automerge` guards.
- **Reusable Release, Security & Internal Workflows ([`references/release-and-maintenance.md`](skills/fastify-workflows/references/release-and-maintenance.md))**: `reusable-release.yml` internals (OIDC provenance, `release` environment gate, shell-safe version bumping), `reusable-ossf-scorecard.yml`, `lock-threads.yml`, and internal workflows (`validate-workflows-files.yml`, `ossf-scorecard.yml`, `release.yml` with `optic-release-automation-action` `sync-semver-tags: true`).
- **Validation, Security & Docs ([`references/validation-and-security.md`](skills/fastify-workflows/references/validation-and-security.md))**: 40-char commit SHA pinning, least-privilege `permissions: {}`, `persist-credentials: false`, shell-injection prevention in `run:` blocks, `action-validator` verification, `README.md` input table sync, and `CONTRIBUTING.md` conventions.

### 5. [`fastify-website`](skills/fastify-website/SKILL.md) — `fastify/website` (`fastify.dev`) Repository
Dedicated skill for maintaining the official [`fastify/website`](https://github.com/fastify/website) (`@fastify/website`, [fastify.dev](https://fastify.dev)) repository:
- **Architecture & Build-Time Data Pipeline ([`references/architecture-and-data-pipeline.md`](skills/fastify-website/references/architecture-and-data-pipeline.md))**: Git-ignored generated data rule (`src/content/docs/`, `public/docs/`, `src/data/*.json`), `scripts/postinstall.mjs`, `scripts/fetch-docs.mjs` (version selection, link/asset rewriting, frontmatter injection), `scripts/build-plugin-list.mjs`, `scripts/fetch-plugin-downloads.mjs`, `scripts/download-benchmarks.mjs`, `scripts/fetch-contributors.mjs` (30-day leaderboard & token-less fallback), `src/content.config.ts`, and `src/data/site.ts`.
- **Components, Pages, Styling & Embedded UI Skills ([`references/components-and-styling.md`](skills/fastify-website/references/components-and-styling.md))**: Astro 7 pages (`src/pages/`, including raw markdown endpoint `docs/[...slug].md.ts`), `withBase()` and `afterVersion()` URL helpers (`src/lib/href.ts`), `DocsShell.astro` & version-scoped Pagefind search (`Search.astro`), Tailwind CSS v4 semantic tokens (`src/styles/tokens.css`), FOUC-free dark/light theme switching, Shiki dual themes, client-side services (`src/services/`), `DOMPurify` XSS sanitization, and integration with the repository's two embedded skills in `.agents/skills/` (`frontend-design` from `anthropics/skills` and `web-design-guidelines` from `vercel-labs/agent-skills`).
- **Validation & Workflows ([`references/validation-and-workflows.md`](skills/fastify-website/references/validation-and-workflows.md))**: Biome linting & formatting (`biome.json`, `npm run lint` / `npm run lint:fix`), `astro check` (`npm run check`), production builds (`npm run build:website`), `.npmrc` supply-chain settings, and GitHub Actions workflows (`deploy-website.yml`, `preview.yml` Netlify staging, and `lock-threads.yml`).

---

## Which Skill to Use

| Target Repository | Implementation / Maintenance Skill | PR Review Skill | Companion Skills |
|-------------------|------------------------------------|-----------------|------------------|
| [`fastify/fastify`](https://github.com/fastify/fastify) | [`fastify-core`](skills/fastify-core/SKILL.md) | [`fastify-code-review`](skills/fastify-code-review/SKILL.md) + `fastify-core` | [`tstyche/tstyche-skills`](https://tstyche.org/guides/agent-skills) |
| `@fastify/*` plugins & `fastify/skeleton` repos | [`fastify-plugins`](skills/fastify-plugins/SKILL.md) | [`fastify-code-review`](skills/fastify-code-review/SKILL.md) + `fastify-plugins` | [`tstyche/tstyche-skills`](https://tstyche.org/guides/agent-skills) |
| [`fastify/workflows`](https://github.com/fastify/workflows) | [`fastify-workflows`](skills/fastify-workflows/SKILL.md) | [`fastify-code-review`](skills/fastify-code-review/SKILL.md) + `fastify-workflows` | — |
| [`fastify/website`](https://github.com/fastify/website) (`fastify.dev`) | [`fastify-website`](skills/fastify-website/SKILL.md) | [`fastify-code-review`](skills/fastify-code-review/SKILL.md) + `fastify-website` | Embedded `.agents/skills/frontend-design` & `.agents/skills/web-design-guidelines` |

---

## Installation

### Using `npx skills` (Multi-Agent Installer)

```bash
npx skills add <owner>/fastify-contribute-skills
```

### Recommended Companion Skills for Type Tests ([TSTyche](https://tstyche.org/guides/agent-skills))

Fastify core and plugin repositories use [TSTyche](https://tstyche.org/) for TypeScript type testing. Install the official TSTyche Agent Skills alongside the Fastify skills:

```bash
npx skills add tstyche/tstyche-skills
```

When installed, `fastify-core` and `fastify-plugins` delegate deep TSTyche authoring and `tsd`-to-TSTyche migration tasks to `tstyche-type-tests`, `tstyche-project-setup`, and `tstyche-migrate-from-tsd`, while retaining built-in TSTyche assertion rules for standalone installations.

### Manual / Symlink Installation by Coding Agent

Copy or symlink the five skills into your coding agent's skill directory:

```bash
# Codex CLI (~/.codex/skills or project .agents/skills)
mkdir -p ~/.codex/skills
ln -s "$(pwd)/skills/fastify-code-review" ~/.codex/skills/fastify-code-review
ln -s "$(pwd)/skills/fastify-core" ~/.codex/skills/fastify-core
ln -s "$(pwd)/skills/fastify-plugins" ~/.codex/skills/fastify-plugins
ln -s "$(pwd)/skills/fastify-workflows" ~/.codex/skills/fastify-workflows
ln -s "$(pwd)/skills/fastify-website" ~/.codex/skills/fastify-website

# Claude Code (~/.claude/skills or project .claude/skills)
mkdir -p ~/.claude/skills
ln -s "$(pwd)/skills/fastify-code-review" ~/.claude/skills/fastify-code-review
ln -s "$(pwd)/skills/fastify-core" ~/.claude/skills/fastify-core
ln -s "$(pwd)/skills/fastify-plugins" ~/.claude/skills/fastify-plugins
ln -s "$(pwd)/skills/fastify-workflows" ~/.claude/skills/fastify-workflows
ln -s "$(pwd)/skills/fastify-website" ~/.claude/skills/fastify-website

# Antigravity (~/.gemini/antigravity-cli/skills or project .agent/skills)
mkdir -p .agent/skills
ln -s "$(pwd)/skills/fastify-code-review" .agent/skills/fastify-code-review
ln -s "$(pwd)/skills/fastify-core" .agent/skills/fastify-core
ln -s "$(pwd)/skills/fastify-plugins" .agent/skills/fastify-plugins
ln -s "$(pwd)/skills/fastify-workflows" .agent/skills/fastify-workflows
ln -s "$(pwd)/skills/fastify-website" .agent/skills/fastify-website
```

---

## Repository Layout

```text
.
├── README.md
└── skills/
    ├── fastify-code-review/
    │   ├── SKILL.md                             # Standardized PR review flow, severity rules & maintainer tone
    │   └── references/
    │       ├── review-flow-and-checklists.md    # Diff inspection commands, governance rules & per-repo checklists
    │       └── tone-and-templates.md            # Fastify maintainer voice, 4-part review template & suggestion blocks
    ├── fastify-core/
    │   ├── SKILL.md                             # Fastify Core (fastify/fastify) skill entry point
    │   └── references/
    │       ├── code.md                          # fastify.js, lib/*, build/* generated validators, symbols, errors/warnings
    │       ├── best-practices.md                # V8 monomorphic shapes, hot-path lifecycle, fastify.d.ts generics, docs/
    │       ├── test.md                          # borp 100% coverage, tspl, bundler (webpack/esbuild), test/types/*.tst.ts
    │       └── workflows.md                     # All 20 fastify/fastify workflows (ci, coverage-nix/win, citgm, backport)
    ├── fastify-plugins/
    │   ├── SKILL.md                             # Fastify Plugins skill (structure, best practices, workflows & releases)
    │   └── references/
    │       ├── code.md                          # Plugin structure, fp(), export triplet, .npmrc, neostandard, @fastify/error
    │       ├── test.md                          # c8 --100 node --test, borp, t.assert, inject() vs listen(), Docker services
    │       ├── type-test.md                     # types/index.d.ts template, module augmentation, TSTyche types/*.tst.ts
    │       └── actions.md                       # Caller workflows (ci.yml, lock-threads.yml, dependabot.yml) & OIDC releases
    ├── fastify-workflows/
    │   ├── SKILL.md                             # Dedicated skill for maintaining the fastify/workflows repository
    │   └── references/
    │       ├── ci-workflows.md                  # plugins-ci.yml, nested-quality.yml, package-manager & 6 service CI workflows
    │       ├── release-and-maintenance.md       # reusable-release.yml, reusable-ossf-scorecard.yml, lock-threads & optic releases
    │       └── validation-and-security.md       # 40-char SHA pinning, shell safety, action-validator, README/CONTRIBUTING rules
    └── fastify-website/
        ├── SKILL.md                             # Dedicated skill for maintaining the fastify/website repository
        └── references/
            ├── architecture-and-data-pipeline.md # Build-time data scripts (fetch-docs, plugins, benchmarks, contributors)
            ├── components-and-styling.md        # Astro 7 pages/components, withBase(), Pagefind, Tailwind v4 tokens, embedded UI skills
            └── validation-and-workflows.md      # Biome linting, astro check, build:website, deploy-website & preview workflows
```

---

## Usage Examples

### 1. `fastify-code-review` + `fastify-core`: Reviewing a PR with Standardized Flow & Tone
```text
Review PR #5812 (or the current branch diff against main) in fastify/fastify using fastify-code-review. Provide inline suggestion blocks and a ready-to-post GitHub review summary.
```

### 2. `fastify-plugins`: Adding a Feature Across Runtime, `.d.ts`, Type Tests & Docs
```text
Add support for the `partitioned` boolean option in @fastify/cookie, updating runtime serialization, types/index.d.ts, TSTyche type tests in types/index.tst.ts, c8 --100 unit tests, and README.md.
```

### 3. `fastify-workflows`: Updating a Reusable CI Workflow in `fastify/workflows`
```text
Add Node.js 25 to the default node-versions matrix in plugins-ci.yml, verify SHA-pinned actions and shell-injection safety, update README.md input tables, and run action-validator.
```

### 4. `fastify-website`: Updating UI or Build-Time Data Scripts in `fastify/website`
```text
Add a new sponsor entry in src/data/site.ts and update the Organizations page layout in fastify/website, following the embedded frontend-design and web-design-guidelines skills, and verify with biome check and astro check.
```

---

## Contribution Policy & Validation Notes

- **Local vs Matrix Validation:** Each skill runs the exact validation commands defined in the target repository (`package.json` scripts). When external services (Redis, PostgreSQL, MySQL, MongoDB, Kafka, Elasticsearch) or multi-OS CI matrices are unavailable locally, the skill reports the exact commands executed and explicitly notes any unverified matrix/service checks.
- **Human-Opened PR Policy:** In accordance with [Fastify's Contribution Policy](https://github.com/fastify/fastify/blob/main/CONTRIBUTING.md), humans are responsible for opening pull requests. These skills prepare verified local patches, conventional PR titles, and structured review or implementation summaries for maintainers.