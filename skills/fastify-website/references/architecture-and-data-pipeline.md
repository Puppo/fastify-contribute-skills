# Fastify Website Architecture & Build-Time Data Pipeline

Detailed reference for the build-time data pipeline (`scripts/`), Astro content collections (`src/content.config.ts`), and static/generated data modules (`src/data/`) in `fastify/website`.

## 1. The Git-Ignored Build Data Rule

`fastify/website` does **not** store Fastify documentation, plugin lists, benchmark results, or contributor leaderboards in git. The `.gitignore` explicitly excludes:

```gitignore
src/content/docs/
public/docs/
src/data/**.json
```

All of these files are generated at build time by running `npm run build:data` (`node scripts/postinstall.mjs`), which is automatically executed before `astro build` in `npm run build:website`.

| Generated Path | Producer Script | Upstream Source |
|----------------|-----------------|-----------------|
| `src/content/docs/<version>/**` | `scripts/fetch-docs.mjs` | `fastify/fastify` release tarballs (`docs/`) |
| `public/docs/<version>/resources/**` | `scripts/fetch-docs.mjs` | `fastify/fastify` release tarballs (`docs/resources/`) |
| `src/data/versions.json` | `scripts/fetch-docs.mjs` | `git ls-remote --tags https://github.com/fastify/fastify.git` |
| `src/data/plugins.json` | `scripts/build-plugin-list.mjs` | `src/content/docs/latest/Guides/Ecosystem.md` |
| `src/data/plugins-by-downloads.json` | `scripts/fetch-plugin-downloads.mjs` | `src/data/plugins.json` + npm Downloads API |
| `src/data/benchmarks.json` | `scripts/download-benchmarks.mjs` | `fastify/benchmarks` (`benchmark-results.json`) + `scripts/frameworks.json` |
| `src/data/contributors.json` | `scripts/fetch-contributors.mjs` | GitHub Search & Reviews API across `fastify/*` repos |

---

## 2. Pipeline Orchestration (`scripts/postinstall.mjs`)

`scripts/postinstall.mjs` runs three concurrent branches via `Promise.all(...)`:

```js
const tasks = [
	downloadBenchmarks(),
	fetchContributors(),
	fetchDocs()
		.then(() => buildPluginList())
		.then(() => fetchPluginDownloads()),
];
```

Notice the strict dependency chain in the third branch:
1. `fetchDocs()` must finish first so `src/content/docs/latest/Guides/Ecosystem.md` exists on disk.
2. `buildPluginList()` parses `Ecosystem.md` and writes `src/data/plugins.json`.
3. `fetchPluginDownloads()` reads `src/data/plugins.json` and writes `src/data/plugins-by-downloads.json`.

---

## 3. Documentation Fetch & Transformation (`scripts/fetch-docs.mjs`)

`scripts/fetch-docs.mjs` mirrors the release-selection and URL-rewriting behavior required for versioned Fastify docs:

1. **Tag Discovery (`listVersions`)**:
   - Runs `git ls-remote --tags --refs https://github.com/fastify/fastify.git` (requires no GitHub token and does not consume API rate limits).
   - Filters `refs/tags/v<major>.<minor>.<patch>` where `major >= DOCS_MIN_MAJOR` (defaults to `3`).
   - **Selection Policy**:
     - **Highest (current) major** (e.g., `5.x`): keeps the **latest patch of every minor** (`v5.10.x`, `v5.9.x`, ..., `v5.0.x`).
     - **Older majors** down to `DOCS_MIN_MAJOR` (e.g., `4.x`, `3.x`): keeps only the **single latest release** of each major (`v4.29.x`, `v3.29.x`).
2. **Tarball Extraction (`downloadAndExtract`)**:
   - Downloads `https://codeload.github.com/fastify/fastify/tar.gz/refs/tags/<tag>` into `.cache/docs/` and extracts only the `docs/` subtree.
3. **Markdown Transformation (`transform` & `rewriteTarget`)**:
   - Strips UTF-8 BOM and the repeated centered `<h1 align="center">Fastify</h1>` header from upstream markdown files.
   - Rewrites relative `.md`/`.mdx` links and Docusaurus-style `/docs/...` paths into base-aware URLs: `${BASE}/docs/${version}/${slug}/#anchor` (stripping trailing `/index` or `/Index`).
   - Rewrites relative `resources/...` image/asset references into `${BASE}/docs/${version}/resources/...` and copies `docs/resources/` to `public/docs/<version>/resources/`.
   - Injects YAML frontmatter required by `src/content.config.ts`:
     ```yaml
     ---
     title: "<Extracted Heading or Filename>"
     section: "<Overview | Guides | Reference>"
     order: <number>
     version: "<version>"
     ---
     ```
   - `order` is `0` for `index.md`, `1..N` for guides listed in `GUIDE_ORDER` (`Getting-Started`, `Recommendations`, `Database`, `Testing`, `Write-Plugin`, `Plugins`, `Serverless`, `Migration-Guide-V5`, `Migration-Guide-V4`, `Migration-Guide-V3`, `Benchmarking`, `Delay-Accept`, `Detecting-When-Clients-Abort`, `Fluent-Schema`, `Prototype-Poisoning`, `Ecosystem`, `Contributing`, `Style-Guide`), and `500` for remaining pages (which sort alphabetically by `title` in `DocsShell.astro`).
4. **Duplicating `latest` & Writing `versions.json`**:
   - Emits the newest version into both `src/content/docs/v<major>.<minor>.x/` and `src/content/docs/latest/`.
   - Writes `src/data/versions.json` with `{ latest, versions: [{ name: "latest", label: "latest (vX.Y.Z)", tag: "vX.Y.Z", isLatest: true }, ...] }`.
5. **Environment Flags & Offline Resilience**:
   - Skips already-fetched version directories unless `FORCE_FETCH=1` is set.
   - Cleans `.cache/docs` after running unless `KEEP_CACHE=1` is set.
   - If `git ls-remote` fails (e.g., offline local development) and `src/content/docs/` is already populated, logs a warning and reuses existing docs instead of crashing.

---

## 4. Ecosystem Plugin Pipeline (`build-plugin-list.mjs` & `fetch-plugin-downloads.mjs`)

- **`scripts/build-plugin-list.mjs`**:
  - Parses `src/content/docs/latest/Guides/Ecosystem.md` using `markdown-it`.
  - Locates `#### [Core](#core)` and `#### [Community](#community)` sections.
  - Extracts each list item's `` [`name`](https://...) description ``, capitalizes the description, tags Core plugins with `official: true`, and writes `src/data/plugins.json` with Core plugins first followed by Community plugins.
  - If `Ecosystem.md` is missing but `src/data/plugins.json` already exists, skips regeneration gracefully.
- **`scripts/fetch-plugin-downloads.mjs`**:
  - Reads `src/data/plugins.json` and queries `https://api.npmjs.org/downloads/point/last-month/<packageName>` with a `350ms` delay between requests and up to 4 retries (respecting HTTP `429` `retry-after`).
  - Defaults unresolved or `404` packages to `0` downloads so builds never abort on npm API hiccups.
  - Sorts plugins descending by `downloads` and writes `src/data/plugins-by-downloads.json` (used by `src/components/index/Ecosystem.astro` to feature the top 6 most-downloaded plugins on the homepage).

---

## 5. Benchmarks Pipeline (`download-benchmarks.mjs` & `frameworks.json`)

- `scripts/frameworks.json` is tracked in git and defines which frameworks from `fastify/benchmarks` to include (e.g., `fastify`, `koa`, `hapi`, `express`, `restify`, `nest`) along with display names and repository URLs.
- `scripts/download-benchmarks.mjs` fetches `https://raw.githubusercontent.com/fastify/benchmarks/main/benchmark-results.json`.
- If the latest `benchmark-results.json` contains invalid/`N/A` metrics for tracked frameworks, it walks the last 10 commits of `fastify/benchmarks` via the GitHub Commits/Git Trees API (`using GH_TOKEN` if provided) until it finds a valid revision.
- Computes `reference` (max req/sec), filters frameworks with `requests > 0`, sorts descending by `requests`, and writes `src/data/benchmarks.json`.

---

## 6. Contributor Leaderboard Pipeline (`fetch-contributors.mjs` & `scripts/contributors/`)

- `scripts/contributors/run.mjs` generates `src/data/contributors.json` representing a rolling 30-day (`WINDOW_DAYS = 30`) activity leaderboard across public `fastify/*` repositories.
- **Scoring Weights (`scripts/contributors/config.mjs`)**:
  - `commits: 1`, `pullRequestsOpened: 3`, `pullRequestsMerged: 2`, `reviews: 2`, `issuesOpened: 1`.
  - Excludes bots/automated accounts (`scripts/contributors/aggregate.mjs`).
- **Token-less Local Build Fallback**:
  - When `GH_TOKEN` / `GITHUB_TOKEN` is **not** set:
    - If `src/data/contributors.json` already exists on disk, it keeps the existing snapshot.
    - If `src/data/contributors.json` does not exist (`ENOENT`), it writes an empty valid `ContributorsData` snapshot (`contributors: []`) so local `npm run build:website` and `npm run check` succeed without requiring a GitHub token.
- `src/data/contributors.ts` imports `~/data/contributors.json` and partitions contributors into two top-10 rosters (`CONTRIBUTOR_LIMIT = 10`):
  - `ACTIVE_MAINTAINERS`: contributors whose login (case-insensitive) is in `LEADS` or `COLLABORATORS` from `~/data/site`.
  - `COMMUNITY_CONTRIBUTORS`: all other contributors.

---

## 7. Tracked Central Site Data (`src/data/site.ts`) & OG Assets (`generate-og-assets.mjs`)

- **`src/data/site.ts`**:
  - `SITE`: site name, tagline, description, URL (`https://fastify.dev`), repo URL, fallback `stars` and `downloads`.
  - `NAV`: top-level navigation links (`/docs/latest`, `/ecosystem/`, `/benchmarks/`, `/organizations/`, `/community/`, `/docs/latest/Reference/LTS/`).
  - `BENCHMARKS` & `BENCHMARK_STATS`: derived at module load from `~/data/benchmarks.json`, including `self` (Fastify), `express`, and formatted multipliers (`multiplier`, `latencyMultiplier`, `throughputMultiplier`).
  - `FEATURES`: 6 homepage feature cards (dynamically interpolating Fastify's `~Xk req/s` from `BENCHMARK_STATS`).
  - `SPONSORS`: array of `{ name, url, image, tier: "collaborator" | "tier_3" | "tier_4", invert?: boolean }`. Logos live in `public/organizations/<image>`.
  - `LEADS`, `COLLABORATORS`, `PAST_COLLABORATORS`: team rosters rendered on the homepage and `/community/`, and used to classify maintainer vs. community contributors.
- **`scripts/generate-og-assets.mjs`** (`npm run generate:og-assets`):
  - Uses `sharp` to rasterize `public/favicon.svg` into `public/og-image.png` (1200x630 dark card), `public/apple-touch-icon.png` (180x180), `public/favicon-32x32.png` (32x32), and a valid ICO container `public/favicon.ico`.

---

## 8. Content Collection Schema (`src/content.config.ts`)

```ts
const docs = defineCollection({
	loader: glob({
		pattern: "**/*.{md,mdx}",
		base: "./src/content/docs",
		generateId: ({ entry }) => entry.replace(/\.(md|mdx)$/i, ""),
	}),
	schema: z.object({
		title: z.string(),
		description: z.string().optional(),
		section: z.string().default("Guides"),
		order: z.number().default(500),
		version: z.string().optional(),
	}),
});
```

- **CRITICAL**: `generateId` strips only the `.md`/`.mdx` extension and **preserves original case and dots** (e.g., `v5.10.x/Reference/Errors`, `latest/Guides/Getting-Started`). Never replace `generateId` with Astro's default slugifier (which lowercases and strips dots), or versioned doc URLs (`v5.10.x`) and rewritten links from `scripts/fetch-docs.mjs` will 404.
