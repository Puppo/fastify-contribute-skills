# Fastify Plugin Workflow Standards & Releases (`actions`)

Load this reference when configuring, reviewing, or updating `.github/workflows/*.yml`, `.github/dependabot.yml`, or release settings in a Fastify plugin repository.

These examples describe repositories using the shared Fastify workflows. Confirm the target checkout's workflows, scripts, and release policy before applying them, including for standalone utilities. Preserve an existing release process unless changing it is part of the task; do not treat differences from a template as policy violations.

> **Note:** This reference covers how **plugin repositories** configure their caller workflows and releases. If you are editing the reusable workflow definitions inside the [`fastify/workflows`](https://github.com/fastify/workflows) repository itself, use the [`fastify-workflows`](../../fastify-workflows/SKILL.md) skill instead.

---

## 1. Plugin Workflow Standard (`fastify/workflows` Callers)

Repositories using [`fastify/workflows`](https://github.com/fastify/workflows) commonly keep thin callers in `.github/workflows/`, pinned to a **full 40-character commit SHA** with a version comment. Inspect specialized build steps and existing caller contracts before changing them:

1. `.github/workflows/ci.yml` — Continuous integration (`plugins-ci.yml` or a `plugins-ci-<service>.yml` variant).
2. `.github/workflows/release.yml` — Manual `workflow_dispatch` release (`reusable-release.yml`).
3. `.github/workflows/lock-threads.yml` — Monthly scheduled locking of inactive closed issues/PRs (`lock-threads.yml`).
4. `.github/workflows/package-manager-ci.yml` *(optional, from `fastify/skeleton`)* — Cross-package-manager validation (`plugins-ci-package-manager.yml`).
5. `.github/dependabot.yml` — Weekly `github-actions` and `npm` updates with 7-day cooldown and grouped devDependencies.

---

## 2. Standard Plugin CI Workflow (`.github/workflows/ci.yml`)

### A. Standard Plugin CI (`plugins-ci.yml`)
Used by plugins that do not require an external database or broker container:

```yaml
name: CI

on:
  push:
    branches:
     - main
     - next
     - 'v*'
    paths-ignore:
      - 'docs/**'
      - '*.md'
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'

# This allows a subsequently queued workflow run to interrupt previous runs
concurrency:
    group: "${{ github.workflow }}-${{ github.event.pull_request.head.label || github.head_ref || github.ref }}"
    cancel-in-progress: true

permissions:
  contents: read

jobs:
  test:
    permissions:
      contents: write
      pull-requests: write
    uses: fastify/workflows/.github/workflows/plugins-ci.yml@ef591e2186785d5ab36b9fe6a79c7ce2f1d94e57 # v7.0.0
    with:
      license-check: true
      lint: true
```

### B. Service-Backed Plugin CI Variants
When a plugin tests against an external backing service, use the matching reusable workflow from `fastify/workflows` in `.github/workflows/ci.yml`:

| Plugin Category | Reusable Workflow to Call | Optional Service Inputs |
| --- | --- | --- |
| **Standard Plugin** (`@fastify/cookie`, `@fastify/cors`, `@fastify/sensible`, `@fastify/multipart`) | `fastify/workflows/.github/workflows/plugins-ci.yml@<sha>` | `lint`, `license-check`, `fastify-dependency-integration`, `node-versions` |
| **Redis** (`@fastify/redis`, `@fastify/rate-limit`) | `fastify/workflows/.github/workflows/plugins-ci-redis.yml@<sha>` | Runs Redis `5, 6, 7, 8` matrix automatically |
| **PostgreSQL** (`@fastify/postgres`) | `fastify/workflows/.github/workflows/plugins-ci-postgres.yml@<sha>` | `postgresql-version` (default `'18'`) |
| **MySQL** (`@fastify/mysql`) | `fastify/workflows/.github/workflows/plugins-ci-mysql.yml@<sha>` | `mysql-version` (default `'9.6'`) |
| **MongoDB** (`@fastify/mongodb`) | `fastify/workflows/.github/workflows/plugins-ci-mongo.yml@<sha>` | Runs MongoDB `8` service container |
| **Kafka** (`@fastify/kafka`) | `fastify/workflows/.github/workflows/plugins-ci-kafka.yml@<sha>` | Runs Zookeeper + Kafka containers |
| **Elasticsearch** (`@fastify/elasticsearch`) | `fastify/workflows/.github/workflows/plugins-ci-elasticsearch.yml@<sha>` | `elasticsearch-version` (default `'9.3.0'`) |

### C. Key `with:` Options for Plugin CI
- `lint: true` — Runs `npm run lint` in the `quality-check` job. Confirm the caller defines that script and preserve its existing lint gate.
- `license-check: true` — Verifies that all production dependencies use permissive SPDX licenses (`0BSD`, `Apache-2.0`, `BlueOak-1.0.0`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`, `MIT`).
- `license-check-allowed-additional: 'Apache*'` — Semicolon-separated list of additional allowed SPDX license patterns when needed by a production dependency.
- `fastify-dependency-integration: true` — Clones `fastify/fastify`, links the plugin via `npm link`, and runs Fastify core's `npm run unit` suite. Enable for plugins that Fastify core itself depends on (`fastify-plugin`, `@fastify/error`, `@fastify/ajv-compiler`, `light-my-request`).
- `node-versions: '["24", "26"]'` — JSON array of Node.js major versions to test against.
- `auto-merge-exclude: 'fastify'` — Semicolon-separated list of packages excluded from automatic Dependabot PR merging.

---

## 3. Plugin Releases (`.github/workflows/release.yml` & Trusted Publishing)

### A. Standard `.github/workflows/release.yml`
For a repository using `fastify/workflows/.github/workflows/reusable-release.yml`, the caller follows this pattern:

```yaml
name: Release the package

on:
  workflow_dispatch:
    inputs:
      semver:
        description: 'Release bump type'
        required: true
        type: choice
        options:
          - patch
          - minor
          - major

permissions: {}

jobs:
  release:
    permissions:
      id-token: write # required for npm provenance via OIDC
      contents: write # required to push the release commit and the tag
    uses: fastify/workflows/.github/workflows/reusable-release.yml@b1a11bce713bc857e52b2231fec3c89071ad3135
    with:
      semver: ${{ inputs.semver }}
```

### B. How the Plugin Release Lifecycle Works
When a maintainer triggers `Release the package` (`workflow_dispatch`) from the GitHub Actions tab and selects `patch`, `minor`, or `major`:
1. **Deployment Environment Gate (`release`):**
   - The job runs inside the `release` GitHub deployment environment (configurable via `with: environment: release`).
   - In repository *Settings -> Environments -> release*, maintainers configure:
     - **Required reviewers** (e.g., `fastify/release` team) so the workflow pauses until an authorized maintainer approves.
     - **Prevent self-review** so the maintainer who triggered the run cannot approve their own release.
     - **Deployment branches** restricted to `main` (or active release branches).
2. **Version Bump & Commit:**
   - Runs `npm version <patch|minor|major> --no-git-tag-version`, stages `package.json`, commits `"Bumped vX.Y.Z"`, and pushes the commit to `origin HEAD`.
3. **Optional Build Step (`release:build`):**
   - Runs `npm install --ignore-scripts --no-audit --no-fund` followed by `npm run --if-present release:build`.
   - If the plugin requires a TypeScript build step before publishing (such as `@fastify/type-provider-typebox`), `package.json#scripts` **must** define `"release:build": "npm run build"` and `"files": ["dist", "index.ts"]`.
4. **OIDC Trusted Publishing to npm (`--provenance`):**
   - Runs `npm publish --provenance --access public`.
   - Authenticates to npm via OpenID Connect (`id-token: write`) configured as an [npm Trusted Publisher](https://docs.npmjs.com/trusted-publishers) bound to the `release` environment—**no `NPM_TOKEN` secret is used**.
   - Ensure `package.json` includes `"publishConfig": { "access": "public" }` for scoped `@fastify/*` packages.
5. **GitHub Release & Release Notes:**
   - Creates tag `vX.Y.Z` and a GitHub Release with `generate_release_notes: true` via `softprops/action-gh-release`.

---

## 4. `.github/workflows/lock-threads.yml` & `package-manager-ci.yml`

### `.github/workflows/lock-threads.yml`
```yaml
name: Lock Threads

on:
  schedule:
    - cron: '0 0 1 * *'
  workflow_dispatch:

concurrency:
  group: lock

permissions:
  contents: read

jobs:
  lock-threads:
    permissions:
      issues: write
      pull-requests: write
    uses: fastify/workflows/.github/workflows/lock-threads.yml@ef591e2186785d5ab36b9fe6a79c7ce2f1d94e57 # v7.0.0
```

### `.github/workflows/package-manager-ci.yml` (Optional, from `fastify/skeleton`)
```yaml
name: package-manager-ci

on:
  push:
    branches:
     - main
     - next
     - 'v*'
    paths-ignore:
      - 'docs/**'
      - '*.md'
  pull_request:
    paths-ignore:
      - 'docs/**'
      - '*.md'

permissions:
  contents: read

jobs:
  test:
    permissions:
      contents: read
    uses: fastify/workflows/.github/workflows/plugins-ci-package-manager.yml@ef591e2186785d5ab36b9fe6a79c7ce2f1d94e57 # v7.0.0
```

---

## 5. `.github/dependabot.yml` Standard for Plugins

This example uses a 7-day cooldown, `chore` commit prefix, `increase-if-necessary` versioning strategy, and grouped ESLint/TypeScript devDependencies. Match the target repository's dependencies and update policy rather than copying unrelated groups or exclusions:

```yaml
version: 2
updates:
  - package-ecosystem: "github-actions"
    directory: "/"
    commit-message:
      prefix: "chore"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
    allow:
      - dependency-name: "*"
        update-types:
          - "version-update:semver-major"
    open-pull-requests-limit: 10

  - package-ecosystem: "npm"
    directory: "/"
    commit-message:
      prefix: "chore"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
    versioning-strategy: "increase-if-necessary"
    allow:
      - dependency-name: "*"
        update-types:
          - "version-update:semver-major"
    ignore:
      # TODO: remove ignore until neostandard support ESLint 10
      - dependency-name: "eslint"
      - dependency-name: "neostandard"
      - dependency-name: "@stylistic/*"
    open-pull-requests-limit: 10
    groups:
      dependencies:
        dependency-type: "production"
      dev-dependencies-eslint:
        patterns:
          - "eslint"
          - "neostandard"
          - "@stylistic/*"
      dev-dependencies-typescript:
        patterns:
          - "@types/*"
          - "tstyche"
          - "typescript"
```

---

## 6. Plugin Workflow & Release Review Checklist

1. **40-Character Commit SHA Pinning:** Ensure every `uses: fastify/workflows/...` and any third-party action in `.github/workflows/*.yml` is pinned to a 40-character commit SHA with a version comment (`# v7.0.0`).
2. **Caller Permissions Match Reusable Requirements:**
   - `ci.yml` `test` job requires `contents: write` and `pull-requests: write` (needed for the nested `automerge` job).
   - `release.yml` `release` job requires `id-token: write` and `contents: write` with top-level `permissions: {}`.
   - `lock-threads.yml` `lock-threads` job requires `issues: write` and `pull-requests: write`.
3. **`--ignore-scripts` Readiness:** Because `plugins-ci.yml` and `reusable-release.yml` run `npm i --ignore-scripts`, plugin repositories must never depend on `prepare` or `postinstall` lifecycle scripts in CI; use `release:build` for pre-publish builds.
