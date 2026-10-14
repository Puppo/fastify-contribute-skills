# Fastify Core GitHub Actions Workflows (`workflows`)

Load this reference when inspecting, debugging, or modifying `.github/workflows/*.yml` in `fastify/fastify`, or when checking how CI gates, PR labels, and ecosystem integration tests run on Fastify core pull requests.

---

## 1. Overview of `fastify/fastify` Workflows (`.github/workflows/`)

Fastify core maintains its own CI and maintenance workflows. The table below maps workflow domains from the repository audit; inspect `.github/workflows/` in the target checkout for current names, triggers, and defaults:

| Category | Workflow File(s) | Purpose |
| --- | --- | --- |
| **Primary CI & Coverage** | `ci.yml`, `coverage-nix.yml`, `coverage-win.yml` | Quality check (`nested-quality.yml`), 100% coverage gate on Linux & Windows, multi-OS/Node unit test matrix, `test-types`, Pino `^9`/`^10` compatibility, Webpack/esbuild bundler tests, and Dependabot automerge. |
| **Alternative Runtimes & Package Managers** | `ci-alternative-runtime.yml`, `integration.yml`, `integration-alternative-runtimes.yml`, `package-manager-ci.yml` | Tests Fastify on NodeSource N\|Solid runtime, production-only installs (`integration/test.sh`), and `pnpm` (v10, v11) + `yarn`. |
| **Ecosystem Canary (CITGM) & Regression** | `citgm.yml`, `citgm-package.yml`, `test-compare.yml` | Runs the test suites of official `@fastify/*` core plugins against the PR branch (`citgm-core-plugins` label), or runs base-vs-PR regression comparison (`test-compare` label). |
| **Documentation & Ecosystem Links** | `md-lint.yml`, `links-check.yml`, `lint-ecosystem-order.yml`, `validate-ecosystem-links.yml`, `website.yml` | Lints markdown (`markdownlint-cli2`), checks internal/external links (`lychee` + `linkinator`), enforces alphabetical order and live repo status in `docs/Guides/Ecosystem.md`, and triggers `fastify/website` deploys. |
| **PR Governance & Maintenance** | `pull-request-title.yml`, `missing_types.yml`, `backport.yml`, `labeler.yml`, `lock-threads.yml` | Enforces Conventional Commit PR titles, opens follow-up issues when PRs labeled `missing-types` merge, auto-backports merged PRs labeled `backport*`, labels PRs, and locks stale threads. |

---

## 2. Primary CI Pipeline (`ci.yml`, `coverage-nix.yml`, `coverage-win.yml`)

### Trigger & Path Filter Rules
- `ci.yml` triggers on `push` (`branches: [main, next, '*.x']`, `paths-ignore: ['docs/**', '*.md']`), `pull_request` (`paths-ignore: ['docs/**', '*.md']`), and `workflow_dispatch` (with a custom `nodejs-version` input for testing Node.js release candidates).
- Markdown-only PRs skip `ci.yml` and instead trigger `md-lint.yml` and `links-check.yml`.

### Job Dependency Graph in `ci.yml`
1. **`quality-check`:**
   - Calls `fastify/workflows/.github/workflows/nested-quality.yml@<sha>` with `license-check: true` and `lint: true` (runs dependency review, production license check, and `npm run lint`).
2. **`coverage-nix` & `coverage-win` (needs `quality-check`):**
   - Calls `./.github/workflows/coverage-nix.yml` (`ubuntu-latest`) and `./.github/workflows/coverage-win.yml` (`windows-latest`).
   - Both run `npm run coverage:ci-check-coverage` (`borp --coverage --check-coverage --lines 100`).
   - **Important:** If line coverage drops below 100% on either Linux or Windows, `test-unit`, `test-types`, and `test-pino-compat` are blocked from running!
3. **`test-unit` (needs `quality-check`, `coverage-nix`, `coverage-win`):**
   - Runs `npm run unit` across `os: [macos-latest, ubuntu-latest, windows-latest]` and `node-version: [24, 26]`.
4. **`test-types` (needs `quality-check`, `coverage-nix`, `coverage-win`):**
   - Runs `npm run test:types` (`tstyche`) on `ubuntu-latest` with `NODE_OPTIONS: no-network-family-autoselection`.
5. **`test-pino-compat` (needs `quality-check`, `coverage-nix`, `coverage-win`):**
   - Tests compatibility across `pino-version: ['^9', '^10']` by running both `npm run unit` and `npm run test:types`.
6. **`package` (needs `test-types`, `test-unit`, `test-pino-compat`):**
   - Runs Webpack (`test/bundler/webpack`) and esbuild (`test/bundler/esbuild`) bundle tests.
7. **`automerge` (needs `test-types`, `test-unit`, `package`):**
   - Automatically merges passing `dependabot[bot]` PRs via `fastify/github-action-merge-dependabot`.

---

## 3. Label-Driven PR Workflows in `fastify/fastify`

Maintainers can trigger specialized workflows on pull requests by applying labels:

1. **`citgm-core-plugins` Label -> `citgm.yml` & `citgm-package.yml`:**
   - Runs the test suites of official `@fastify/*` ecosystem plugins by linking the Fastify PR checkout (`npm link` -> `npm link fastify` -> `npm test` in each plugin repo).
   - After completion, the `remove-label` job automatically removes the `citgm-core-plugins` label from the PR.
2. **`test-compare` Label -> `test-compare.yml`:**
   - Runs `nearform-actions/github-action-test-compare` with `testCommand: 'npm run test:ci'` to verify that a new test fails on the base branch and passes on the PR branch.
3. **`missing-types` Label -> `missing_types.yml`:**
   - When a PR with the `missing-types` label is merged, automatically creates a new issue titled `add missing types to [<PR title>]` with labels `["good-first-issue", "typescript"]`.
4. **`backport <branch>` Label -> `backport.yml`:**
   - Uses `tibdex/backport` when a merged PR has a `backport` label to open a backport PR against the target maintenance branch (e.g., `5.x`, `4.x`).

---

## 4. PR Title & Documentation Workflows

1. **PR Title Regex (`pull-request-title.yml`):**
   - Uses `fastify/action-pr-title` with regex:
     ```text
     /^(\[Backport [0-9]+?\.x\]\s)?(build|chore|ci|docs|feat|types|fix|perf|refactor|style|test)(?:\([^\):]*\))?!?:\s/
     ```
   - Allowed types: `build`, `chore`, `ci`, `docs`, `feat`, `types`, `fix`, `perf`, `refactor`, `style`, `test` (with optional `[Backport X.x] ` prefix).
2. **Markdown & Links (`md-lint.yml`, `links-check.yml`, `lint-ecosystem-order.yml`):**
   - `md-lint.yml` runs `./node_modules/.bin/markdownlint-cli2` on PRs touching `**/*.md`.
   - `links-check.yml` runs `lycheeverse/lychee-action` (offline internal link check across `./**/*.md`) and `JustinBeckwith/linkinator-action` (external link check on modified markdown files excluding `docs/Guides/Ecosystem.md`).
   - `lint-ecosystem-order.yml` runs `.github/scripts/lint-ecosystem.js` when `docs/Guides/Ecosystem.md` is modified.

---

## 5. Workflow Security & Maintenance Rules for `fastify/fastify`

1. **40-Character Commit SHA Pinning:** Every external `uses:` action in `.github/workflows/*.yml` must be pinned to a full 40-character commit SHA with a version comment (`# v7.0.0`).
2. **Least-Privilege Permissions & `persist-credentials: false`:** Keep top-level `permissions: contents: read`, declare explicit job-level permissions, and pass `persist-credentials: false` on every `actions/checkout` step.
3. **Cross-Platform Path Safety in Tests:** Because `coverage-win.yml` and `test-unit` run on `windows-latest`, never hardcode POSIX `/` file paths or `file://` URL assumptions in unit tests without `node:path` normalization.
