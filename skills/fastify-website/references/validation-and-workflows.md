# Fastify Website Validation & GitHub Actions Workflows

Reference for linting (`biome`), type/template checking (`astro check`), local builds, and GitHub Actions workflows in `fastify/website`.

## 1. Linting & Formatting with Biome (`biome.json`)

Unlike Fastify core and plugin repositories (which use ESLint + `neostandard`), `fastify/website` uses **Biome** (`@biomejs/biome`) for formatting, import sorting, and linting across `.mjs`, `.ts`, `.astro`, `.css`, and `.json` files.

### Commands
- `npm run lint` -> runs `biome check` (verifies formatting, lint rules, and import organization).
- `npm run lint:fix` -> runs `biome check --write` (auto-fixes formatting, import order, and safe lint rules).

### Key `biome.json` Rules & Conventions
- **Indentation & Quotes**: Biome defaults (tab indentation, double quotes, semicolons in JS/TS/MJS).
- **Tailwind CSS v4 Directives**: `"css": { "parser": { "tailwindDirectives": true }, "formatter": { "enabled": true } }` enables parsing of `@import "tailwindcss";` and `@theme inline`.
- **Import Organization**: `"organizeImports": "on"` enforces sorted imports.
- **`.astro` Overrides**:
  - In `**/*.astro` files, `a11y.useAnchorContent`, `correctness.noUnusedImports`, `correctness.noUnusedVariables`, and `suspicious.noExplicitAny` are turned off because Astro frontmatter variables/imports are referenced in the template section outside Biome's JS scope analyzer.
- **Excluded Generated Paths**:
  - `biome.json` explicitly ignores `!**/dist`, `!**/build`, `!**/.astro`, `!**/.cache`, `!src/content/docs`, `!public/docs`, `!src/data/benchmarks.json`, `!src/data/plugins.json`, `!src/data/plugins-by-downloads.json`, `!src/data/versions.json`, `!src/data/contributors.json`, `!**/package-lock.json`, `!**/*.svg`, and `!skills-lock.json`.
- **Inline Suppression**: When `!important` is required in CSS (such as overriding Shiki's inline light-mode styles in `src/styles/prose.css`), annotate with `/* biome-ignore lint/complexity/noImportantStyles: override Shiki inline styles */`.

---

## 2. Type & Astro Checking (`npm run check`)

- Command: `npm run check` (`astro check` via `@astrojs/check` and `typescript`).
- `tsconfig.json` extends `"astro/tsconfigs/strict"` with `"strictNullChecks": true`, `"baseUrl": "."`, and `"paths": { "~/*": ["src/*"] }`.
- **Prerequisite (`npm run build:data`)**:
  - `src/data/site.ts`, `src/data/contributors.ts`, `src/pages/ecosystem.astro`, `src/components/VersionSelect.astro`, and `src/components/VersionBanner.astro` statically import generated JSON files (`~/data/benchmarks.json`, `~/data/contributors.json`, `~/data/plugins.json`, `~/data/versions.json`).
  - Always run `npm run build:data` before `npm run check` if those git-ignored JSON files are not yet present in `src/data/`.

---

## 3. Building & Previewing the Website

| Script | Command | Description |
|--------|---------|-------------|
| `npm run build:data` | `node scripts/postinstall.mjs` | Runs `downloadBenchmarks()`, `fetchContributors()`, and `fetchDocs() -> buildPluginList() -> fetchPluginDownloads()`. |
| `npm run build:website` | `npm run build:data && astro build` | Full production build: generates all data/docs, compiles static HTML/CSS/JS into `./build`, runs `astro-inference` and `ViteImageOptimizer`, and builds the Pagefind search index in `./build/pagefind/`. |
| `npm run dev` / `npm start` | `astro dev` | Starts the local Astro dev server (requires `npm run build:data` to have been run at least once; note that Pagefind search requires a production build). |
| `npm run preview` | `astro preview` | Serves the production `./build` output locally (including working Pagefind search). |
| `npm run generate:og-assets` | `node scripts/generate-og-assets.mjs` | Regenerates `public/og-image.png`, `public/apple-touch-icon.png`, `public/favicon-32x32.png`, and `public/favicon.ico` from `public/favicon.svg`. |

### Environment Variables for `build:data` / `build:website`
- `GH_TOKEN` / `GITHUB_TOKEN`: Optional for local builds, recommended in CI. Used by `scripts/download-benchmarks.mjs` (higher GitHub API rate limits) and `scripts/fetch-contributors.mjs` (to query 30-day org activity; without it, `fetch-contributors.mjs` reuses the existing snapshot or writes an empty contributor list).
- `FORCE_FETCH=1`: Forces `scripts/fetch-docs.mjs` to re-download and re-transform all Fastify version tarballs even if `src/content/docs/<version>` already exists.
- `KEEP_CACHE=1`: Retains downloaded tarballs in `.cache/docs/` after `fetch-docs.mjs` finishes.
- `DOCS_MIN_MAJOR`: Minimum Fastify major version to fetch (defaults to `3`).

### Production Preview Checks for UI & Routing Changes

After a successful production build, run the checkout's preview command and exercise affected behavior in a real browser. Select checks matching the diff; build success alone does not establish interaction correctness.

- **Versioned search:** Open search on `latest` and a supported older version. Confirm results belong to the selected version and their links resolve. Use the production preview because Pagefind requires the built index.
- **Keyboard access:** Open the search dialog from its trigger and shortcut, navigate by keyboard, close with Escape, and verify focus returns to the trigger. Check visible focus, labels, and mobile navigation where affected.
- **Themes:** Switch between dark and light, reload, and check persistence, readable contrast, and code blocks. Check reduced-motion behavior when changing animations.
- **Version switching and docs links:** Switch the same page across available versions, verify the selected version/banner, anchors, canonical/edit links, and the raw Markdown endpoint. For pages absent in another version, verify the checkout's intended missing-page behavior rather than inventing a fallback.
- **Base paths:** Check internal links, assets, Markdown routes, and Pagefind requests with a non-root base using the existing base configuration in an isolated checkout. Confirm case and dots in versioned paths survive rewriting.
- **Runtime evidence:** Record the preview URL, tested routes, browser observations, and relevant console/network errors. If browser execution is unavailable, report it as `Not run` with the reason; static UI audits and successful builds remain separate evidence.

---

## 4. Repository Security & Package Settings (`.npmrc` & `.nvmrc`)

- `.nvmrc`: Pins Node.js `24`.
- `.npmrc`:
  ```ini
  ignore-scripts=true
  min-release-age=7
  ```
  - Supply-chain protection: blocks automatic lifecycle scripts during `npm install` / `npm ci` (which is why `scripts/postinstall.mjs` is invoked explicitly via `npm run build:data`) and enforces a 7-day minimum release age on npm packages.

---

## 5. GitHub Actions Workflows (`.github/workflows/`)

### A. Production Deployment (`.github/workflows/deploy-website.yml`)
- **Triggers**:
  - `push` to `main`, `next`, `v*` (ignoring `.github/**` and `.vscode/**`).
  - `workflow_dispatch` (manual production deploy).
  - `workflow_call` (reusable workflow called by `fastify/fastify`'s `.github/workflows/notify-website-update.yml` whenever a new Fastify release is published or `docs/**` changes on `fastify/fastify` `main`).
- **Jobs**:
  1. `build-and-upload`:
     - Checks out repo (`persist-credentials: false`), sets up Node.js from `.nvmrc` with npm cache, runs `npm ci --ignore-scripts`.
     - Configures GitHub Pages (`actions/configure-pages`), runs `npm run build:website` (`timeout-minutes: 30`, `GH_TOKEN: ${{ github.token }}`), and uploads `./build` via `actions/upload-pages-artifact`.
  2. `deploy-to-production`:
     - Depends on `build-and-upload`, uses concurrency group `${{ github.ref_name }}-production` (`cancel-in-progress: true`), and deploys to GitHub Pages (`environment: production`) via `actions/deploy-pages` with `pages: write` and `id-token: write` permissions.

### B. Pull Request Preview & CI Check (`.github/workflows/preview.yml`)
- **Triggers**: `pull_request` targeting `main`.
- **Jobs**:
  1. `build-and-upload` (`environment: sandbox`):
     - Checks out the PR branch, sets up Node.js from `.nvmrc`, runs `npm ci --ignore-scripts`.
     - **Runs Linting**: `npm run lint` (`biome check`).
     - Resolves the latest `fastify/fastify` release tag (`oprypin/find-latest-tag`) and caches `scripts/releases`.
     - **Builds Website**: `npm run build:website` (`GH_TOKEN: ${{ github.token }}`), verifying that the full data pipeline and `astro build` succeed on the PR.
     - Uploads `./build` via `actions/upload-pages-artifact`.
  2. `deploy-to-staging` (`environment: staging`):
     - Downloads and extracts the `github-pages` artifact into `./website` and deploys a staging preview to Netlify using `netlify/actions/cli` (`NETLIFY_SITE_ID`, `NETLIFY_AUTH_TOKEN`).

### C. Thread Locking (`.github/workflows/lock-threads.yml`)
- Monthly cron (`0 0 1 * *`) and `workflow_dispatch` calling `fastify/workflows/.github/workflows/lock-threads.yml` (pinned to a 40-char commit SHA) to lock inactive closed issues and PRs.

### Workflow Maintenance Rules
- Always pin third-party and reusable GitHub Actions to full **40-character commit SHAs** with a version comment (e.g., `actions/checkout@3d3c42e5aac5ba805825da76410c181273ba90b1 # v7.0.1`).
- Keep top-level `permissions: contents: read` and scope elevated permissions (`pages: write`, `id-token: write`, `pull-requests: write`) to the specific job that needs them.
