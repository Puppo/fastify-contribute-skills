# Workflow Authoring, Security, Validation & Documentation in `fastify/workflows` (`validation-and-security`)

Load this reference when validating workflow changes, auditing GitHub Actions security in `fastify/workflows`, or updating `README.md` / `CONTRIBUTING.md`.

---

## 1. Security & Hardening Standards in `fastify/workflows`

Every workflow in `.github/workflows/` must adhere to OSSF Scorecard and Fastify supply-chain security rules:

1. **Full 40-Character Commit SHA Pinning:**
   - Pin every external action (`actions/checkout`, `actions/setup-node`, `actions/dependency-review-action`, `actions/upload-artifact`, `github/codeql-action/upload-sarif`, `ossf/scorecard-action`, `pnpm/action-setup`, `softprops/action-gh-release`, `fastify/github-action-merge-dependabot`, `nearform-actions/optic-release-automation-action`, `jsumners/lock-threads`, `asdf-vm/actions/install`) to a **40-character commit SHA** with a trailing version comment:
     ```yaml
     uses: actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1
     ```
   - Never use mutable tags (`@v4`, `@main`) for external actions in `.github/workflows/*.yml`.
   - For local reusable workflows within `fastify/workflows` (`nested-quality.yml`, `reusable-ossf-scorecard.yml`), reference the relative path `./.github/workflows/<name>.yml`.
2. **Least-Privilege `permissions`:**
   - Set top-level `permissions: {}` on all reusable workflows (`on: workflow_call`) and `permissions: contents: read` on standalone workflows.
   - Declare explicit minimal `permissions:` on every individual job (`contents: read` for test/lint/license jobs; `pull-requests: write` and `contents: write` only on `automerge`).
3. **`persist-credentials: false` on Checkouts:**
   - Every `actions/checkout` step in CI, quality, and scorecard workflows must include:
     ```yaml
     with:
       persist-credentials: false
     ```
   - The **only** exception is `reusable-release.yml`, which explicitly sets `persist-credentials: true` and `fetch-depth: 0` so it can push the version-bump commit and tag.
4. **No Direct `${{ inputs.* }}` Interpolation in `run:` Shell Scripts:**
   - Interpolating `${{ inputs.foo }}` directly inside a `run:` script allows untrusted input values to break quotes or execute arbitrary shell commands.
   - Always bind inputs to environment variables (`env:`) and reference `"$ENV_VAR"` in shell commands (see `ALLOWED_ADDITIONAL` in `nested-quality.yml` and `BUMP_TYPE` in `reusable-release.yml`).
5. **`--ignore-scripts` by Default:**
   - Use `npm i --ignore-scripts` (or `pnpm install --ignore-scripts` / `yarn install --ignore-engines --ignore-scripts`) unless a specific database workflow (`plugins-ci-postgres.yml`, `plugins-ci-kafka.yml`) requires native addon compilation.

---

## 2. Validating Workflow Changes Locally

Because `fastify/workflows` has no `npm test` script, validate changes using these steps:

1. **Run `action-validator` (if installed locally) or YAML Schema Checks:**
   - In CI, `validate-workflows-files.yml` runs:
     ```bash
     find .github/workflows -type f \( -iname \*.yaml -o -iname \*.yml \) \
       | xargs -I {} action-validator --verbose {}
     ```
   - If `action-validator` is installed in the environment (`which action-validator`), run it across `.github/workflows/*.yml`.
   - If `action-validator` is not installed locally, parse all `.github/workflows/*.yml` files with Python (`yaml.safe_load`) or Ruby/Node YAML parsers to verify valid YAML syntax, and verify that every `needs:` job reference, `matrix` variable, and `inputs.*` reference matches defined keys.
2. **Verify Matrix & Expression Types:**
   - Confirm `node-versions` defaults are valid JSON arrays of strings (`'["24", "26"]'`) so `fromJson(inputs.node-versions)` evaluates cleanly.
   - Confirm `automerge` `needs:` arrays match the exact job IDs in the workflow (`[fastify-dependency-integration, quality-check, test]` in `plugins-ci.yml` vs `[quality-check, test]` in service workflows).

---

## 3. Documentation & Contribution Conventions (`README.md` & `CONTRIBUTING.md`)

1. **Keep `README.md` Synchronized with Workflow Inputs:**
   - Whenever you add, rename, or change the default of an input in `plugins-ci*.yml` or `reusable-release.yml`, update the corresponding **Inputs** markdown table in `README.md` in the same patch.
2. **Style Rules from `CONTRIBUTING.md`:**
   - **Language:** Write markdown documentation and inline YAML comments in **American English**.
   - **Headings:** Use **sentence-style capitalization** for titles and headings (only the first letter of the heading and proper nouns like `CI`, `Node.js`, `GitHub`, `OSSF Scorecard` are capitalized—e.g., `### Enable workflow Linter job`, `### Restricting who can release`).
   - **Branch & Commit Conventions:** Follow the Feature Branch Workflow (`feat/` branches targeting `main`) and Conventional Commits (`feat:`, `fix:`, `ci:`, `docs:`, `chore:`).
