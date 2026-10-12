# Repository Discovery & Scope Verification

Before proposing code changes, running commands, or issuing review findings, discover the target repository's scope status, conventions, and local tooling from checked-in files. Never assume every Fastify repository shares `fastify/fastify` core's scripts or directory structure.

## 1. Organization Scope & Active Maintenance Check

When operating as a Fastify maintainer, verify that the target repository is in scope and actively maintained:

1. **Identify the repository and last commit date:**
   - Inspect `git remote -v` and `package.json#name` / `package.json#repository`.
   - Run `git log -1 --format="%H %cs"` and `git log -1 --since="2 years ago" --format="%H %cs"` on the target base branch (`main`, `master`, or maintenance branch such as `v4.x`).
2. **Enforce the 2-Year Activity Cutoff:**
   - If `git log -1 --since="2 years ago" --format="%H %cs"` produces no output (or the repository is marked archived/deprecated in `README.md`), stop and warn the maintainer that the repository is **inactive (>2 years without commits)** before proceeding.
3. **Enforce the Official Ecosystem Plugin Rule:**
   - Distinguish **Fastify plugins** (packages registered via `fastify.register()`, wrapped with `fastify-plugin`, or implementing `FastifyTypeProvider`) from **standalone shared libraries** (listed in "Non-Plugin Active Organization Repositories" below, several of which also use the `@fastify/*` npm scope such as `@fastify/error`, `@fastify/busboy`, `@fastify/deepmerge`, `@fastify/send`, `@fastify/restartable`).
   - If the repository is a Fastify plugin or type provider under the `fastify` GitHub organization, it **must** appear in the official Core Plugins list on [https://fastify.dev/ecosystem/](https://fastify.dev/ecosystem/) (listed below).
   - Plugin repositories in the `fastify` organization that are **not** on this list (for example, `fastify-fetch`, `fastify-opensearch`, `fastify-type-provider-zod`, `fastify-valkey-glide`, or archived plugins) are **unmanned** and out of scope for standard maintainer automation unless the user explicitly overrides this rule.

### Canonical `fastify.dev/ecosystem` Core Plugins (60 Repositories)

| Repository (`fastify/<repo>`) | Package Name |
| --- | --- |
| `aws-lambda-fastify` | `@fastify/aws-lambda` |
| `csrf-protection` | `@fastify/csrf-protection` |
| `fastify-accepts` | `@fastify/accepts` |
| `fastify-accepts-serializer` | `@fastify/accepts-serializer` |
| `fastify-auth` | `@fastify/auth` |
| `fastify-autoload` | `@fastify/autoload` |
| `fastify-awilix` | `@fastify/awilix` |
| `fastify-basic-auth` | `@fastify/basic-auth` |
| `fastify-bearer-auth` | `@fastify/bearer-auth` |
| `fastify-caching` | `@fastify/caching` |
| `fastify-circuit-breaker` | `@fastify/circuit-breaker` |
| `fastify-compress` | `@fastify/compress` |
| `fastify-cookie` | `@fastify/cookie` |
| `fastify-cors` | `@fastify/cors` |
| `fastify-elasticsearch` | `@fastify/elasticsearch` |
| `fastify-env` | `@fastify/env` |
| `fastify-etag` | `@fastify/etag` |
| `fastify-express` | `@fastify/express` |
| `fastify-flash` | `@fastify/flash` |
| `fastify-formbody` | `@fastify/formbody` |
| `fastify-funky` | `@fastify/funky` |
| `fastify-helmet` | `@fastify/helmet` |
| `fastify-hotwire` | `@fastify/hotwire` |
| `fastify-http-proxy` | `@fastify/http-proxy` |
| `fastify-jwt` | `@fastify/jwt` |
| `fastify-kafka` | `@fastify/kafka` |
| `fastify-mongodb` | `@fastify/mongodb` |
| `fastify-multipart` | `@fastify/multipart` |
| `fastify-mysql` | `@fastify/mysql` |
| `fastify-nextjs` | `@fastify/nextjs` |
| `fastify-oauth2` | `@fastify/oauth2` |
| `fastify-passport` | `@fastify/passport` |
| `fastify-postgres` | `@fastify/postgres` |
| `fastify-rate-limit` | `@fastify/rate-limit` |
| `fastify-redis` | `@fastify/redis` |
| `fastify-reply-from` | `@fastify/reply-from` |
| `fastify-request-context` | `@fastify/request-context` |
| `fastify-response-validation` | `@fastify/response-validation` |
| `fastify-routes` | `@fastify/routes` |
| `fastify-routes-stats` | `@fastify/routes-stats` |
| `fastify-schedule` | `@fastify/schedule` |
| `fastify-secure-session` | `@fastify/secure-session` |
| `fastify-sensible` | `@fastify/sensible` |
| `fastify-static` | `@fastify/static` |
| `fastify-swagger` | `@fastify/swagger` |
| `fastify-swagger-ui` | `@fastify/swagger-ui` |
| `fastify-throttle` | `@fastify/throttle` |
| `fastify-type-provider-json-schema-to-ts` | `@fastify/type-provider-json-schema-to-ts` |
| `fastify-type-provider-typebox` | `@fastify/type-provider-typebox` |
| `fastify-url-data` | `@fastify/url-data` |
| `fastify-vite` | `@fastify/vite` |
| `fastify-websocket` | `@fastify/websocket` |
| `fastify-zipkin` | `@fastify/zipkin` |
| `middie` | `@fastify/middie` |
| `one-line-logger` | `@fastify/one-line-logger` |
| `otel` | `@fastify/otel` |
| `point-of-view` | `@fastify/view` |
| `session` | `@fastify/session` |
| `sse` | `@fastify/sse` |
| `under-pressure` | `@fastify/under-pressure` |

### Non-Plugin Active Organization Repositories

Non-plugin repositories in `https://github.com/fastify` are in scope when unarchived and committed to within the last 2 years, including:
- **Core framework:** `fastify`
- **Shared libraries & plumbing:** `accept-negotiator`, `ajv-compiler`, `avvio`, `busboy`, `csrf`, `deepmerge`, `env-schema`, `fast-content-type-parse`, `fast-json-stringify`, `fast-json-stringify-compiler`, `fast-uri`, `fastify-error`, `fastify-plugin`, `fluent-json-schema`, `forwarded`, `json-schema-ref-resolver`, `light-my-request`, `merge-json-schemas`, `openapi-schema-diff`, `process-warning`, `proxy-addr`, `restartable`, `safe-regex2`, `secure-json-parse`, `send`, `server`
- **Tooling & CLI:** `create-fastify`, `eslint-plugin-fastify`, `fastify-citgm`, `fastify-cli`, `fastify-snippet`, `pre-commit`, `releasify`, `skeleton`, `tsconfig`
- **Docs, Benchmarks & Examples:** `benchmarks`, `demo`, `example`, `graphics`, `help`, `social-posts`, `website`
- **Workflows & Org Automation:** `.github`, `action-pr-title`, `fastify-template-gitpod`, `gh-issues-finder`, `github-action-merge-dependabot`, `org-admin`, `workflows`

---

## 2. Discovery Evidence Matrix

Inspect these files in the target checkout and record the discovered evidence before selecting commands or editing files:

| Dimension | Files to Inspect | What to Record |
| --- | --- | --- |
| **Policies & Instructions** | `CONTRIBUTING.md`, `.github/CONTRIBUTING.md`, `README.md`, `SECURITY.md`, `AGENTS.md` | AI contribution rules (e.g., human-opened PR policy in `fastify/fastify`), commit/PR title format (`action-pr-title`), release-note expectations. |
| **Package & Exports** | `package.json`, `pnpm-workspace.yaml`, `tsconfig.json` | Package name, module format (`"type": "module"` vs CommonJS default), `main`, `exports`, `types`/`typings` entry points, and whether a build step (`npm run build`) generates `dist/` or validators. |
| **Generated Files** | `package.json#scripts` (`build*`, `test:validator:integrity`), `.gitignore` | Scripts that regenerate committed files (e.g., `lib/config-validator.js` and `lib/error-serializer.js` in `fastify/fastify`). |
| **Package Manager** | `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `.github/workflows/*.yml` | Active package manager (`npm` is standard across most Fastify repos; check lockfiles and CI workflow steps). |
| **Linter & Formatter** | `package.json#scripts` (`lint*`), `eslint.config.*`, `.eslintrc*`, `biome.json` | Linter tool and exact script (`npm run lint`, `npm run lint:fix`, `neostandard`, `@fastify/eslint-config`, `biome check` in `website`, or `markdownlint-cli2`). |
| **Runtime Test Runner** | `package.json#scripts` (`test*`, `unit*`, `coverage*`), `package.json#devDependencies`, `test/` | Runner (`borp`, `node --test`, `tap`, `tape`, `playwright-test`), assertion library (`@matteo.collina/tspl`, `node:assert`, `tap`), coverage wrapper (`c8 --100`, `borp -C --check-coverage`). |
| **Type Test Runner** | `package.json#scripts` (`test:types`, `test:typescript`), `tstyche.config.*`, `tsconfig.json`, `test/types/`, `types/` | Whether type tests use **TSTyche** (`tstyche`), `tsd`, `tsc --project ...`, or `attw --pack .`, and the file pattern (`*.test-d.ts`, `*.tst.ts`). |
| **CI & Services** | `.github/workflows/*.yml`, `docker-compose*.yml` | Reusable workflows (`fastify/workflows/.github/workflows/plugins-ci.yml` or `ci.yml`), Node.js version matrix, OS matrix, and external service containers (Redis, PostgreSQL, MySQL, MongoDB, Kafka, Elasticsearch). |
| **Documentation** | `README.md`, `docs/**/*.md` | Where public options, error codes, deprecation warnings, and examples are documented. |

---

## 3. Handling Edge Cases & Missing Tooling

1. **No `package.json` (e.g., `fastify/workflows`, `fastify/.github`, `fastify/graphics`):**
   - Mark JavaScript, TypeScript, runtime tests, and type tests as **Not Applicable** with the reason (`No package.json; repository contains only GitHub Actions workflows / assets`).
   - Route directly to `references/ci-docs.md`.
2. **No Type Tests or Declarations (e.g., `fastify/website`, `fastify/benchmarks`):**
   - If `package.json` has no `types`/`typings` entry, no `test/types/` or `types/*.test-d.ts` / `*.tst.ts` files, and no `tstyche`/`tsd` script, mark **Type Tests** as **Not Applicable** unless the task explicitly adds TypeScript declarations. Do not invent a type-test suite for a repo that does not ship types.
3. **Different Script Names Across Fastify Repos:**
   - Fastify core uses `npm run unit` and `npm run test:types`.
   - Most Fastify plugins and libraries (`fastify-sensible`, `fastify-multipart`, `fastify-cookie`, `fastify-plugin`, `fast-json-stringify`) use `npm run test:unit` and `npm run test:typescript`.
   - `fastify-type-provider-typebox` uses `npm run build`, `npm run test:node`, and `npm run test:types`.
   - Always invoke the exact script defined in the target repository's `package.json`.
4. **Conflicting Evidence or Missing Dependencies:**
   - If `node_modules` is missing or a required external service (such as a local Redis, PostgreSQL, or Kafka broker) is unavailable, record the exact command attempted and the environment limitation rather than guessing or fabricating passing test results.
