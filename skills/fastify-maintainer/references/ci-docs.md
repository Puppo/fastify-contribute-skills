# CI, Documentation & Contribution Policy Guidelines

Load this reference when inspecting or modifying `.github/workflows/`, `package.json` scripts/engines/dependencies, `README.md`, `docs/`, error/deprecation tables, or PR/contribution metadata.

## 1. Continuous Integration (`.github/workflows/`)

### Shared `fastify/workflows` & Repository Matrices
Most Fastify repositories delegate CI to reusable workflows in [`fastify/workflows`](https://github.com/fastify/workflows) (such as `.github/workflows/plugins-ci.yml`) or define a `ci.yml` workflow:
1. **Verify Job Reachability & Triggers:**
   - Check `on: [push, pull_request]` path filters (`paths-ignore`, `branches`) so required test, lint, and type-check jobs are not accidentally skipped.
2. **Node.js, TypeScript & OS Matrices:**
   - Check the Node.js version matrix in `.github/workflows/ci.yml` and `package.json#engines`. Do not use Node.js runtime APIs unavailable in the oldest supported Node.js version on the target branch without a feature check or fallback.
3. **External Service Containers in CI:**
   - Database and messaging plugins (`@fastify/redis`, `@fastify/postgres`, `@fastify/mysql`, `@fastify/mongodb`, `@fastify/kafka`, `@fastify/elasticsearch`) configure Docker services in `.github/workflows/ci.yml` or `docker-compose.yml`. When changing driver versions or connection options, verify compatibility with the CI service container image and environment variables.
4. **Linter & Integrity Gates:**
   - **`neostandard` / ESLint:** Almost all active Fastify codebases run `npm run lint` (`eslint` configured with `neostandard`).
   - **Markdown Linting:** `fastify/fastify` runs `npm run lint:markdown` (`markdownlint-cli2`).
   - **Biome:** `fastify/website` runs `npm run lint` (`biome check`).
   - **Generated File Integrity:** `fastify/fastify` runs `npm run test:validator:integrity` in CI to ensure `lib/config-validator.js` and `lib/error-serializer.js` match their generators.
5. **Workflow Supply-Chain Security & Trusted Publishing:**
   - Pin third-party GitHub Actions to full 40-character commit SHAs (with a version comment) when the repository follows SHA-pinning conventions, and preserve explicit least-privilege `permissions:` blocks.
   - Preserve reusable release workflows (`fastify/workflows/.github/workflows/release.yml` with npm trusted publishing / `id-token: write`) and `.npmrc` security settings (such as `min-release-age`).
6. **Honest Local vs Remote CI Reporting:**
   - Local validation runs against a single OS and Node.js version. Explicitly note when a change affects cross-platform path handling (`win32` vs `posix`), Node.js version-specific behavior, or external service matrices that could not be executed locally.

---

## 2. Documentation (`README.md` & `docs/`)

1. **Public Options, Defaults & Signatures:**
   - Every new or modified plugin/server option, decorator, or hook must be documented in `README.md` (for plugins/libraries) or `docs/Reference/*.md` (for `fastify/fastify`).
   - Ensure documented default values, option types, and code snippets match both the `.js` runtime implementation and `.d.ts` declarations.
2. **Error & Deprecation Tables:**
   - In `fastify/fastify`, adding or changing a `FST_ERR_*` code in `lib/errors.js` requires updating `docs/Reference/Errors.md`.
   - Adding or changing a `FSTDEP*` warning code in `lib/warnings.js` requires updating `docs/Reference/Warnings.md` (or the relevant LTS/migration guide).
3. **Documentation Code Examples:**
   - Keep code examples runnable and aligned with the repository's module format and current Fastify major version (`fastify.listen({ port: 3000 })` object signature rather than removed legacy positional arguments).

---

## 3. Contribution Policy & PR Preparation (`CON-004`)

1. **Human-Opened PR Policy:**
   - Inspect `CONTRIBUTING.md` in the target repository. `fastify/fastify` requires humans to open pull requests and take full responsibility for any AI-assisted code. Never automatically open or push remote PRs unless explicitly authorized by the maintainer and permitted by the repository's policy.
2. **PR Title Conventions (`fastify/action-pr-title`):**
   - Fastify repositories enforce Conventional Commits on PR titles via `fastify/action-pr-title` (e.g., `fix: handle stream abort in multipart parser`, `feat: add partitioned cookie option`, `types: fix reply schema narrowing`, `docs: document FST_ERR_...`, `test: ...`, `chore: ...`).
   - Always include a compliant PR title and concise summary in your implementation output.
