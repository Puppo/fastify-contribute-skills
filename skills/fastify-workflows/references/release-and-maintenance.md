# Reusable Release, Security & Internal Workflows in `fastify/workflows` (`release-and-maintenance`)

Load this reference when reviewing or modifying `reusable-release.yml`, `reusable-ossf-scorecard.yml`, `lock-threads.yml`, or `fastify/workflows`'s own internal workflows (`release.yml`, `ossf-scorecard.yml`, `validate-workflows-files.yml`, and `.github/dependabot.yml`).

---

## 1. `reusable-release.yml` (Organization-Wide npm OIDC Release Workflow)

`reusable-release.yml` is called by `.github/workflows/release.yml` across Fastify plugin and library repositories to bump versions, publish to npm with provenance, and create GitHub releases.

### Input Contract
| Input | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `semver` | `string` | `true` | — | Release bump type: `patch`, `minor`, or `major`. |
| `node-version` | `string` | `false` | `'lts/*'` | Node.js version used to build and publish. |
| `runs-on` | `string` | `false` | `'ubuntu-latest'` | Runner used to publish the package. |
| `environment` | `string` | `false` | `'release'` | GitHub deployment environment that gates the release with required reviewers. |

### Critical Security & Implementation Invariants in `reusable-release.yml`
1. **Deployment Environment Gate (`environment: ${{ inputs.environment }}`):**
   - Runs inside the `release` deployment environment by default so consuming repositories can require team approval (e.g., `fastify/release`) and prevent self-review before any step executes.
2. **Permissions:**
   - Top-level `permissions: {}`.
   - Job-level `permissions:` grants **only** `id-token: write` (for npm OIDC provenance) and `contents: write` (to push the release commit and tag).
3. **Shell-Injection-Safe Version Bump:**
   - Passes `${{ inputs.semver }}` through job `env: BUMP_TYPE: ${{ inputs.semver }}`.
   - Validates `$BUMP_TYPE` against `patch|minor|major` in a `case` statement before running `npm version`.
   - Runs `npm version "$(printf '%s' "$BUMP_TYPE")" --no-git-tag-version`, stages tracked files with `git add -u`, commits `"Bumped v${NEW_VERSION#v}"`, and writes sanitized `NEW_VERSION` to `$GITHUB_ENV`.
4. **Checkout & Node Setup Rules:**
   - `actions/checkout` **must** set `fetch-depth: 0` and `persist-credentials: true` because the workflow pushes the release commit back to `origin HEAD`.
   - `actions/setup-node` **must** set `registry-url: 'https://registry.npmjs.org'` and `package-manager-cache: false` (never use caching in release builds).
5. **Install, Build & Publish Sequence:**
   - `npm install --ignore-scripts --no-audit --no-fund`
   - `npm run --if-present release:build` (allows TypeScript/bundled packages to build `dist/` even with `--ignore-scripts`)
   - `git push origin HEAD`
   - `npm publish --provenance --access public` (uses npm OIDC trusted publishing—never require `NPM_TOKEN`)
   - `softprops/action-gh-release` with `tag_name: v${{ env.NEW_VERSION }}`, `target_commitish: ${{ github.ref_name }}`, `generate_release_notes: true`, and `make_latest: true`.

---

## 2. `reusable-ossf-scorecard.yml` & `lock-threads.yml`

### `reusable-ossf-scorecard.yml`
- **Trigger:** `on: workflow_call` with optional boolean input `publish_results` (default `true`).
- **Permissions:** Top-level `permissions: {}`; job `analysis` grants `contents: read`, `id-token: write`, `security-events: write`.
- **Steps:**
  1. `actions/checkout` with `persist-credentials: false`.
  2. `ossf/scorecard-action` writing `results.sarif` (`publish_results: ${{ inputs.publish_results && github.ref_name == github.event.repository.default_branch }}`).
  3. `actions/upload-artifact` (`retention-days: 5`).
  4. `github/codeql-action/upload-sarif`.

### `lock-threads.yml`
- **Trigger:** `on: workflow_call` (no inputs).
- **Permissions:** Top-level `permissions: {}`; job `lock-threads` grants `issues: write` and `pull-requests: write`.
- **Step:** Runs `jsumners/lock-threads` with `issue-inactive-days: '90'`, `pr-inactive-days: '90'`, and `exclude-any-issue-labels: 'discussion,good first issue,help wanted'`.

---

## 3. Internal Workflows of `fastify/workflows` Itself

Unlike the reusable `workflow_call` workflows above, these three workflows run directly on the `fastify/workflows` repository:

1. **`validate-workflows-files.yml` (Workflow Linter CI):**
   - Triggers on `push` to `main` and `pull_request` when `.github/workflows/*.yml` changes.
   - Installs `action-validator 0.5.3` via `asdf-vm/actions/install` and runs:
     ```bash
     find .github/workflows -type f \( -iname \*.yaml -o -iname \*.yml \) \
       | xargs -I {} action-validator --verbose {}
     ```
2. **`ossf-scorecard.yml`:**
   - Triggers on `branch_protection_rule`, `push` to `main` (`paths-ignore: ["**/*.md"]`), weekly `schedule` (`cron: "21 17 * * 0"`), and `workflow_dispatch`.
   - Calls `./.github/workflows/reusable-ossf-scorecard.yml` with `publish_results: true`.
3. **`release.yml` (Releasing `fastify/workflows` Itself):**
   - Triggers on `workflow_dispatch` (with `inputs.semver` default `'patch'`) and `pull_request: types: [closed]`.
   - Uses `nearform-actions/optic-release-automation-action` with `sync-semver-tags: true`.
   - **Why `release.yml` differs from `reusable-release.yml`:** `fastify/workflows` is not published to npm; instead, consumers reference it via Git tags (`@v7`, `@v7.1`, `@v7.1.0`, or commit SHA). `optic-release-automation-action` opens a release PR to `main` and, when merged/closed, automatically updates floating major/minor tags (`v7` and `v7.1`) alongside the exact semver tag (`v7.1.0`).
4. **`.github/dependabot.yml`:**
   - Monitors `package-ecosystem: github-actions` monthly with a 7-day cooldown, `commit-message` prefix `ci` (`include: scope`), and groups `actions/*`, `github/*`, and `octokit/*` into `github-owned`.
