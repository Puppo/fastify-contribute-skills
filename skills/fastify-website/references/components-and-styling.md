# Fastify Website Components, Pages & Styling

Reference for Astro pages (`src/pages/`), UI components (`src/components/`), URL/markdown helpers (`src/lib/`), client-side services (`src/services/`), and the Tailwind CSS v4 design token system (`src/styles/`) in `fastify/website`.

## 1. Pages & Routing (`src/pages/`)

| Route File | URL Path | Purpose & Key Details |
|------------|----------|-----------------------|
| `src/pages/index.astro` | `/` | Marketing homepage composed of section components in `src/components/index/` (`Hero`, `Why`, `CoreFeatures`, `QuickStart`, `Benchmarks`, `Ecosystem`, `Team`, `Cta`). |
| `src/pages/docs/index.astro` | `/docs/` | Redirects immediately to `withBase("/docs/latest/")` via `Astro.redirect(target)` + `<meta http-equiv="refresh">`. |
| `src/pages/docs/[...slug].astro` | `/docs/<version>/<slug>/` | Renders every entry in the `docs` collection inside `DocsShell.astro` (`slug = doc.id.replace(/\/(index\|Index)$/, "")`). Extracts `version = doc.id.split("/")[0]` and `minutesRead` from `remarkPluginFrontmatter`. |
| `src/pages/docs/[...slug].md.ts` | `/docs/<version>/<slug>.md` | Static `APIRoute` (`GET`) returning the raw Markdown (`doc.body`) with `Content-Type: text/markdown; charset=utf-8` (linked as "View as Markdown" in `DocsShell.astro`). |
| `src/pages/ecosystem.astro` | `/ecosystem/` | Renders all Core (`official: true`) and Community plugins from `~/data/plugins.json` using `EcosystemCard.astro` with client-side search (`[data-plugin-search]`) matching `data-name` and `data-desc`. |
| `src/pages/benchmarks.astro` | `/benchmarks/` | Renders Requests/sec, Latency, and Throughput comparisons from `BENCHMARK_STATS` using `BenchStat.astro` and `BenchBars.astro`. |
| `src/pages/organizations.astro` | `/organizations/` | Displays `collaborator`, `tier_4` (strategic sponsors), and `tier_3` (supporting sponsors) from `SPONSORS` using `OrganizationLogo.astro`. |
| `src/pages/community.astro` | `/community/` | Displays top active maintainers & community contributors (`~/data/contributors`), team rosters, community channels, and an interactive client-side Good First Issues filter (`[data-good-first-issues]`). |
| `src/pages/contributors.astro` | `/contributors/` | Full rolling 30-day activity leaderboard for active maintainers and community contributors with scoring methodology weights. |
| `src/pages/404.astro` | `/404.html` | Custom 404 error page wrapped in `BaseLayout.astro`. |

---

## 2. Base-Aware URLs & Version Helpers (`src/lib/href.ts`)

`astro.base.config.mjs` defines the single source of truth for Astro's `base` path (`base: "/"` in production, configurable for subpath deployments).

- **`withBase(path: string): string`**:
  - Prefixes any root-absolute path (starting with `/`) with `import.meta.env.BASE_URL` while collapsing duplicate slashes.
  - Passes external URLs (`https://...`), `mailto:`, and `#fragment` links through unchanged.
  - **Rule**: Every internal `<a href="/...">`, `<link href="/...">`, `<img src="/...">`, and client-side asset URL (such as `withBase('/pagefind/pagefind.js')`) MUST use `withBase(...)`.
- **`afterVersion(docId: string): string`**:
  - Strips the leading version segment (`"v5.9.x/Reference/Errors"` -> `"Reference/Errors"`, `"latest"` -> `""`).
  - Used by `DocsShell.astro` and `VersionBanner.astro` when switching versions or building the upstream GitHub "Edit this page" URL (`${SITE.repo}/edit/main/docs/${afterVersion(current) || "index"}.md`).

---

## 3. Documentation Shell & Version-Scoped Search

### `src/components/DocsShell.astro`
- Renders the 3-column documentation layout:
  1. **Left Sidebar**: `VersionSelect.astro` (`[data-version-switch]`), Search trigger button (`[data-search-open]` with `⌘K` badge), and `DocsNav.astro` grouped by `SECTION_ORDER` (`Overview: 0`, `Guides: 1`, `Reference: 2`).
  2. **Main Article**:
     - Mobile navigation `<details>` drawer + `VersionSelect`.
     - `VersionBanner.astro`: displayed whenever `version !== latestEntry.name`, linking directly to the equivalent page in `latest`.
     - Breadcrumb trail, reading time (`minutesRead`), "View as Markdown" link (`/docs/${markdownSlug}.md`), and `<article class="prose-fastify" data-pagefind-body data-pagefind-filter={`version:${version}`} data-version={version}>`.
     - Prev/Next page cards and "Edit this page on GitHub" footer link.
  3. **Right Sidebar (`TableOfContents.astro`)**:
     - Renders `h2` and `h3` headings (`depth >= 2 && depth <= 3`).
     - Client-side `IntersectionObserver` (`rootMargin: "0px 0px -70% 0px"`) highlights the active heading (`[data-toc-link]`).
- **Version Switcher Navigation**:
  - When `[data-version-switch]` changes to `nextVersion`, client script navigates to `${base}docs/${nextVersion}/${rest}` (or `${base}docs/${nextVersion}/` if on index).

### `src/components/Search.astro` (Pagefind)
- Uses a native `<dialog data-search-dialog>` opened via `[data-search-open]` or `Cmd+K` / `Ctrl+K`.
- Reads the active documentation version from `document.querySelector('[data-pagefind-body][data-version]')?.dataset.version || 'latest'`.
- Lazy-loads Pagefind on first open via `import(/* @vite-ignore */ withBase('/pagefind/pagefind.js'))`.
- Queries `pf.debouncedSearch(term, { filters: { version: [currentVersion] } }, 220)` so searches on a `v4.29.x` page only return `v4.29.x` results, and searches outside docs default to `latest`.
- Sanitizes result titles with `DOMPurify.sanitize(...)` before inserting into `innerHTML`.

---

## 4. Markdown Processing & Copy-to-Clipboard Pipeline

Configured in `astro.config.mjs` for both Markdown and MDX:
- **`src/lib/remark-reading-time.mjs`**: Computes reading time via `reading-time` and `mdast-util-to-string`, attaching `data.astro.frontmatter.minutesRead`.
- **`src/lib/rehype-code-copy.mjs`**: Wraps every `<pre>` element in `<div class="docs-pre-wrap" data-copy-root data-code-lines="single|multiple">`.
- **`src/components/DocsShell.astro` & `src/lib/copy.ts`**:
  - `DocsShell.astro` clones a `<template data-docs-copy-template>` containing `<CopyButton class="docs-copy-btn" iconOnly />` into every `article.prose-fastify .docs-pre-wrap`.
  - `initCopyButtons()` in `src/lib/copy.ts` (initialized once in `BaseLayout.astro`) listens for clicks on `[data-copy-btn]`, resolves text from `data-copy-text`, `data-copy-target`, or the nearest `[data-copy-root] pre code`, writes to `navigator.clipboard`, toggles `[data-copy-idle]` / `[data-copy-done]` for `1600ms`, and announces `"Copied"` via an `aria-live="polite"` screen-reader region.

---

## 5. Design Tokens, Themes & Tailwind CSS v4 (`src/styles/`)

`src/styles/global.css` imports Tailwind CSS v4 (`@import "tailwindcss";`) followed by four modular stylesheets:

1. **`src/styles/tokens.css`**:
   - **Fonts**: `--font-display` (`Space Grotesk Variable`), `--font-sans` (`Inter Variable`), `--font-mono` (`JetBrains Mono Variable`).
   - **Default Dark Identity (`:root`)**:
     - `--bg: #0a0b0d`, `--surface: #101419`, `--surface-2: #161c23`, `--surface-3: #1d242c`
     - `--border: #232b33`, `--border-strong: #313b45`
     - `--fg: #e9ecef`, `--fg-soft: #c2cad2`, `--muted: #8b95a1`
     - `--amber: #c99a4a`, `--amber-soft: #e8c078`, `--amber-ink: #c99a4a`, `--teal: #e8d9b0`, `--red: #ff5d5d`
   - **Light Theme Override (`:root[data-theme="light"]`)**:
     - `--bg: #f6f4ef`, `--surface: #ffffff`, `--surface-2: #f1ede4`, `--surface-3: #e8e3d8`
     - `--border: #e3ddd0`, `--border-strong: #d3cbba`
     - `--fg: #14181c`, `--fg-soft: #333c45`, `--muted: #5c6670`
     - `--amber-ink: #8c6b2e` (darkened for WCAG text contrast on light surfaces)
   - Exposes all semantic tokens to Tailwind via `@theme inline` (`bg-bg`, `bg-surface`, `bg-surface-2`, `border-border`, `border-border-strong`, `text-fg`, `text-fg-soft`, `text-muted`, `text-amber`, `text-amber-ink`, `text-teal`, `text-red`).
2. **`src/styles/base.css`**:
   - Base typography, `:focus-visible` (`2px solid var(--amber)`), and `::selection`.
3. **`src/styles/utilities.css`**:
   - Signature utility classes: `.bg-telemetry` (56px grid backdrop), `.glow-amber` (radial spotlight), `.eyebrow` (uppercase mono label in `--amber-ink`), `.text-velocity` (amber-to-teal gradient text), `.card` (translucent surface with border & backdrop blur), `.animate-ticker`.
   - `@media (prefers-reduced-motion: reduce)` disables smooth scroll, `.animate-ticker`, `[data-streak]`, and transitions.
4. **`src/styles/prose.css`**:
   - `.prose-fastify` styles for rendered documentation markdown (headings, diamond list bullets, tables, blockquotes, code blocks).
   - **Shiki Dual-Theme Switching**: Shiki is configured in `astro.config.mjs` with `themes: { light: "github-light", dark: "github-dark" }` and `defaultColor: "dark"`. When `[data-theme="light"]` is active on `<html>`, `prose.css` overrides `.astro-code` and `.astro-code span` with `var(--shiki-light)` and `var(--shiki-light-bg)`.
5. **No-FOUC Theme Initialization (`src/layouts/BaseLayout.astro`)**:
   - An inline `<script is:inline>` in `<head>` checks `localStorage.getItem('theme') === 'light'` and sets `document.documentElement.setAttribute('data-theme', 'light')` before first paint.
   - `[data-theme-toggle]` buttons toggle `data-theme="light"` and persist to `localStorage`.

---

## 6. Client-Side Services (`src/services/`) & XSS Sanitization

- **`src/services/github.ts` (`getRepoStars`)**:
  - Fetches `https://api.github.com/repos/${owner}/${repo}`, caches in `sessionStorage` (`github-stars:${owner}/${repo}`), and populates `[data-github-stars]` elements in `BaseLayout.astro` formatted with `format-si-unit`.
- **`src/services/npm.ts` (`getMonthlyDownloads`)**:
  - Fetches `https://api.npmjs.org/downloads/point/last-month/${packageName}`, caches in `sessionStorage` (`npm-downloads:${packageName}`), and populates `[data-npm-downloads]` elements.
- **`src/services/gfi.ts` (`getGoodFirstIssues`)**:
  - Fetches `https://goodfirstissue.fastify.io/api/find-issues?org=${org}`, caches in `sessionStorage` (`fastify-good-first-issues:${org}`), and powers the interactive project filter on `/community/`.
- **XSS Sanitization Rules**:
  - In `.astro` components using `set:html` on markdown/external strings (e.g., `EcosystemCard.astro`), always sanitize with `DOMPurify.sanitize(await marked.parseInline(...))` from `isomorphic-dompurify`.
  - In client-side `<script>` blocks building `innerHTML` from external APIs (e.g., `Search.astro` or `community.astro`), always sanitize with `DOMPurify.sanitize(...)` or escape HTML entities (`escapeHtml`).

---

## 7. Project-Embedded UI/UX Skills (`frontend-design` & `web-design-guidelines`)

`fastify/website` checks in two dedicated Agent Skills under `.agents/skills/` (with symlinks in `.claude/skills/` and hashes locked in `skills-lock.json`):

1. **`frontend-design` (`.agents/skills/frontend-design/SKILL.md` — `anthropics/skills`)**:
   - **When to Load**: Whenever creating new UI sections, redesigning pages/components, adjusting visual hierarchy, or authoring interface copy.
   - **Core Directives Applied to `fastify/website`**:
     - Ground visual choices in Fastify's domain (high-throughput, low-overhead Node.js telemetry and velocity: `.bg-telemetry`, `.glow-amber`, `.eyebrow`, `VelocityMeter.astro`, `BenchBars.astro`).
     - Maintain the 3-role typography system (`--font-display` Space Grotesk for headings, `--font-sans` Inter for prose/body, `--font-mono` JetBrains Mono for metrics, eyebrows, tags, and code).
     - Use structural devices (eyebrows like `// benchmarks`, `[tier 04]` markers, dividers) only where they encode real hierarchy or sequence.
     - Exercise restraint: keep surfaces disciplined (`--surface`, `--border`), respect `prefers-reduced-motion: reduce`, ensure visible keyboard `:focus-visible`, and write concise, active-voice, user-centered copy.
2. **`web-design-guidelines` (`.agents/skills/web-design-guidelines/SKILL.md` — `vercel-labs/agent-skills`)**:
   - **When to Load**: Whenever reviewing UI pull requests, auditing accessibility, or checking `.astro` components against UX best practices.
   - **How It Works**:
     - Fetches the latest rules from `https://raw.githubusercontent.com/vercel-labs/web-interface-guidelines/main/command.md`, checks the modified `.astro`/`.ts`/`.css` files against all rules (semantic HTML, keyboard navigation, focus states, responsive layout, form/input labels, loading/empty/error states), and reports findings in `file:line` format.
   - **Note on `skills-lock.json`**: Do not manually edit `skills-lock.json` or remove `.agents/skills/` / `.claude/skills/` unless updating the embedded skills via `npx skills`. `skills-lock.json` is excluded from Biome formatting in `biome.json`.
