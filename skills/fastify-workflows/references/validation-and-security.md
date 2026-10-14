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

Inspect the checkout's validation workflow and scripts. Report each level separately:

1. **YAML syntax:** Parse changed workflows with a YAML 1.2-compatible parser so the `on` key remains a string. If using a YAML 1.1 parser such as default PyYAML, account for its boolean-key coercion before inspecting triggers. Successful parsing proves syntax only.
2. **Workflow schema:** Run the configured `action-validator` command when available. A common CI command is:
   ```bash
   find .github/workflows -type f \( -iname \*.yaml -o -iname \*.yml \) \
     | xargs -I {} action-validator --verbose {}
   ```
   If the validator is unavailable, mark schema validation `Not run`; parsing or manual inspection does not substitute for it.
3. **Contracts and expressions:** Inspect `needs`, inputs, permissions, matrix JSON, and guards in context, including reusable callers. For example, confirm `node-versions` is a JSON array of supported version strings and optional jobs can be skipped without allowing failures through the automerge guard. Static validation does not prove runtime expression evaluation or service readiness.
4. **Execution evidence:** Use existing CI runs tied to the reviewed revision. Record run links, events, caller inputs, and observed results. If representative execution is unavailable, mark it `Not run` and state the remaining uncertainty. Dispatching workflows, changing labels, or running a publish path requires authorization for that external action.

### Representative Caller Scenarios

Select scenarios affected by the workflow change; use static inspection when execution would require an unauthorized external action:

| Scenario | Expected contract |
| --- | --- |
| Minimal caller | Omitted optional inputs retain documented defaults; required inputs and permissions are explicit. |
| Overrides | Custom Node matrices, lint/license flags, and affected service versions reach the correct jobs and nested workflows. |
| Fork pull request | Test/validation jobs tolerate the available token/secrets; untrusted code does not gain privileged execution and automerge remains disabled. |
| Optional integration job | An intentional skip does not block successful checks; failure or cancellation does not permit automerge. |
| Release caller | Required permissions, environment protection, trusted-publishing configuration, and build hooks remain compatible. Inspect gates without publishing to prove a change. |

Maintain input documentation alongside contract changes. Report YAML, schema, contract inspection, and CI execution as distinct evidence in the review matrix.

---

## 3. Documentation & Contribution Conventions (`README.md` & `CONTRIBUTING.md`)

1. **Keep `README.md` Synchronized with Workflow Inputs:**
   - Whenever you add, rename, or change the default of an input in `plugins-ci*.yml` or `reusable-release.yml`, update the corresponding **Inputs** markdown table in `README.md` in the same patch.
2. **Style Rules from `CONTRIBUTING.md`:**
   - **Language:** Write markdown documentation and inline YAML comments in **American English**.
   - **Headings:** Use **sentence-style capitalization** for titles and headings (only the first letter of the heading and proper nouns like `CI`, `Node.js`, `GitHub`, `OSSF Scorecard` are capitalized—e.g., `### Enable workflow Linter job`, `### Restricting who can release`).
   - **Branch & Commit Conventions:** Follow the Feature Branch Workflow (`feat/` branches targeting `main`) and Conventional Commits (`feat:`, `fix:`, `ci:`, `docs:`, `chore:`).
