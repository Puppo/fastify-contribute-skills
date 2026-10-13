# Reusable CI & Service Workflows in `fastify/workflows` (`ci-workflows`)

Load this reference when reviewing or modifying `plugins-ci.yml`, `nested-quality.yml`, `plugins-ci-package-manager.yml`, or any of the 6 service-container CI workflows (`plugins-ci-redis.yml`, `plugins-ci-postgres.yml`, `plugins-ci-mysql.yml`, `plugins-ci-mongo.yml`, `plugins-ci-kafka.yml`, `plugins-ci-elasticsearch.yml`).

---

## 1. Shared Input Contract Across `plugins-ci*.yml`

All `plugins-ci*.yml` workflows (except `plugins-ci-package-manager.yml`, which only accepts `node-versions`) share these core `workflow_call` inputs and defaults:

| Input Name | Type | Required | Default | Purpose |
| --- | --- | --- | --- | --- |
| `auto-merge-exclude` | `string` | `false` | `'fastify'` | Semicolon-separated list of npm packages excluded from Dependabot automerge. |
| `license-check` | `boolean` | `false` | `false` | Passed to `nested-quality.yml` to run production license checks. |
| `license-check-allowed-additional` | `string` | `false` | `''` | Semicolon-separated additional SPDX license identifiers passed to `nested-quality.yml`. |
| `lint` | `boolean` | `false` | `false` | Passed to `nested-quality.yml` to run `npm run lint`. |
| `node-versions` | `string` | `false` | `'["24", "26"]'` | JSON array string parsed via `fromJson(inputs.node-versions)` for the Node.js test matrix. |

### Workflow-Specific Additional Inputs
- `plugins-ci.yml`:
  - `fastify-dependency-integration` (`boolean`, default `false`): Clones `fastify/fastify`, runs `npm link "$PACKAGE_NAME"`, and runs `npm run unit` in Fastify core.
- `plugins-ci-elasticsearch.yml`:
  - `elasticsearch-version` (`string`, `required: true`, default `'9.3.0'`): Docker image tag for `elasticsearch:${{ inputs.elasticsearch-version }}`.
- `plugins-ci-mysql.yml`:
  - `mysql-version` (`string`, `required: false`, default `'9.6'`): Docker image tag for `mysql:${{ inputs.mysql-version }}`.
- `plugins-ci-postgres.yml`:
  - `postgresql-version` (`string`, `required: false`, default `'18'`): Docker image tag for `postgres:${{ inputs.postgresql-version }}-alpine`.

> **Cross-File Update Rule:** Whenever you update the default `node-versions` array or upgrade `actions/checkout`, `actions/setup-node`, or `fastify/github-action-merge-dependabot`, check **all 8** `plugins-ci*.yml` files (plus `nested-quality.yml`) so no workflow drifts behind.

---

## 2. `nested-quality.yml` (Quality Gate Helper)

Every `plugins-ci*.yml` workflow starts with a `quality-check` job that calls `./.github/workflows/nested-quality.yml`:

```yaml
  quality-check:
    uses: ./.github/workflows/nested-quality.yml
    permissions:
      contents: read
    with:
      license-check: ${{ inputs.license-check }}
      license-check-allowed-additional: ${{ inputs.license-check-allowed-additional }}
      lint: ${{ inputs.lint }}
```

`nested-quality.yml` defines three jobs:
1. **`dependency-review`:**
   - Runs only on pull requests (`if: github.event_name == 'pull_request'`).
   - Uses `actions/dependency-review-action` with `contents: read`.
2. **`license-check`:**
   - Runs when `!failure() && !cancelled() && inputs.license-check == true`.
   - Installs dependencies with `npm i --ignore-scripts` and runs:
     ```bash
     npx license-checker --production --summary --onlyAllow="0BSD;Apache-2.0;BlueOak-1.0.0;BSD-2-Clause;BSD-3-Clause;ISC;MIT;$ALLOWED_ADDITIONAL"
     ```
   - Passes `inputs.license-check-allowed-additional` via `env: ALLOWED_ADDITIONAL` (never interpolate `${{ inputs.* }}` directly into the `run:` shell string!).
3. **`linter`:**
   - Runs when `!failure() && !cancelled() && inputs.lint == true`.
   - Runs `npm i --ignore-scripts` and `npm run lint` on `node-version: lts/*`.

---

## 3. Core `plugins-ci.yml` Architecture

1. **`test` Matrix:**
   - `os: [macos-latest, ubuntu-latest, windows-latest]`
   - `node-version: ${{ fromJson(inputs.node-versions) }}`
   - Legacy macOS exclusions for Node `10, 12, 14, 16` are preserved for older maintenance branches that pass custom `node-versions`.
   - Steps: `actions/checkout` (`persist-credentials: false`) -> `actions/setup-node` (`check-latest: true`) -> `npm i --ignore-scripts` -> `npm test`.
2. **`fastify-dependency-integration` Job:**
   - Runs on `ubuntu-latest` with `node-version: lts/*` when `!failure() && !cancelled() && inputs.fastify-dependency-integration == true`.
   - Links the current plugin package (`npm link` + `PACKAGE_NAME=$(node -p "require('./package.json').name")`), shallow-clones `fastify/fastify`, installs with `npm i --ignore-scripts`, links `"$PACKAGE_NAME"`, and runs `npm run unit`.
3. **`automerge` Job:**
   - Needs `[fastify-dependency-integration, quality-check, test]`.
   - Guard condition in `plugins-ci.yml`:
     ```yaml
     if: >
       always() &&
       needs.quality-check.result == 'success' &&
       needs.test.result == 'success' &&
       (needs.fastify-dependency-integration.result == 'success' || needs.fastify-dependency-integration.result == 'skipped') &&
       github.event_name == 'pull_request' &&
       github.event.pull_request.head.repo.full_name == github.repository &&
       github.event.pull_request.user.login == 'dependabot[bot]'
     ```
   - Grants `pull-requests: write` and `contents: write` and invokes `fastify/github-action-merge-dependabot` with `target: major` and `exclude: ${{ inputs.auto-merge-exclude }}`.

---

## 4. Service-Container CI Workflows (`plugins-ci-<service>.yml`)

All service-container workflows run on `ubuntu-latest` (because GitHub Actions Docker service containers require Linux runners) and configure `needs: [quality-check, test]` on their `automerge` job:

| Workflow File | Docker Service(s) & Matrix | Install & Environment Specifics |
| --- | --- | --- |
| `plugins-ci-redis.yml` | `redis:${{ matrix.db }}` with `matrix.db: [5, 6, 7, 8]`, port `6379:6379`, `--entrypoint redis-server` | `npm i --ignore-scripts` -> `npm test` |
| `plugins-ci-postgres.yml` | `postgres:${{ inputs.postgresql-version }}-alpine` (default `'18'`), port `5432:5432`, healthcheck `pg_isready` | Runs `npm i` (native `pg` bindings), installs `node-gyp`, `libpq-dev`, `postgresql-client`, sets up `.pgpass`, and creates test `users` table before `npm test`. |
| `plugins-ci-mysql.yml` | `mysql:${{ inputs.mysql-version }}` (default `'9.6'`), port `3306:3306`, `MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'`, `MYSQL_DATABASE: mysql`, healthcheck `mysqladmin ping` | `npm i --ignore-scripts` -> `npm test` |
| `plugins-ci-mongo.yml` | `mongo:${{ matrix.db }}` with `matrix.db: [8]`, port `27017:27017` | `npm i --ignore-scripts` -> `npm test` |
| `plugins-ci-kafka.yml` | `confluentinc/cp-zookeeper:7.4.3` (`2181:2181`) + `confluentinc/cp-kafka:7.4.3` (`9092:9092`, `--link zookeeper`) | Runs `npm i` (for native Kafka addons if any) -> `npm test` |
| `plugins-ci-elasticsearch.yml` | `elasticsearch:${{ inputs.elasticsearch-version }}` (default `'9.3.0'`), ports `9200:9200` & `9300:9300`, `xpack.security.enabled: false`, `discovery.type: single-node`, healthcheck `curl http://localhost:9200/_cluster/health` | `npm i --ignore-scripts` -> `npm test` |

---

## 5. `plugins-ci-package-manager.yml`

Tests plugin compatibility with alternative package managers on `ubuntu-latest` across `node-version: ${{ fromJson(inputs.node-versions) }}`:
- **`pnpm` job:** Matrix `pnpm-version: [10, 11]` using `pnpm/action-setup`, runs `pnpm install --ignore-scripts` and `pnpm run test`.
- **`yarn` job:** Installs Yarn via `https://yarnpkg.com/install.sh`, runs `yarn install --ignore-engines --ignore-scripts`, `yarn`, and `yarn run test`.
