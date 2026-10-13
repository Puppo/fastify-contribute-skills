# Fastify PR Review Flow & Technical Checklists

Detailed reference for inspecting diffs, verifying Fastify organization governance, and running repository-specific technical checks during a pull request review.

## 1. Diff & Metadata Inspection Commands

Always inspect the full context of the change before writing findings:

### When Reviewing a GitHub PR (`gh` CLI available)
```bash
# View PR title, description, labels, base/head branches, and author
gh pr view <number> --json title,body,baseRefName,headRefName,labels,author,files

# Inspect the unified diff with line numbers
gh pr diff <number>

# Check CI status checks on the PR
gh pr checks <number>
```

### When Reviewing a Local Branch or Commit Range
```bash
# Confirm working tree is clean (do not modify tracked files during review)
git status --short

# Inspect commit messages and changed files against base branch (e.g., main or next)
git log --oneline origin/main..HEAD
git diff --stat origin/main...HEAD

# Inspect the full diff
git diff origin/main...HEAD
```

- **Read surrounding context**: Never judge a hunk in isolation. Read the surrounding function, hook registration, `.d.ts` interface, and test file to verify whether invariants (like encapsulation, error codes, or `t.plan(N)` counts) hold.

---

## 2. Organization-Wide Governance & PR Hygiene Checklist

Check these rules on every Fastify PR regardless of repository:

| Check | Rule | Severity if Violated |
|-------|------|----------------------|
| **PR Title (`fastify/action-pr-title`)** | Must follow Conventional Commits: `feat:`, `fix:`, `docs:`, `chore:`, `test:`, `refactor:`, `perf:`, `ci:`, `build:` (optional scope e.g. `feat(types): ...`). | `[Blocking]` (CI check fails on merge) |
| **`package.json` Version Unchanged** | Contributors must **never** bump `"version"` in `package.json` inside a PR (`CONTRIBUTING.md`: formal releases are declared by maintainers via release workflows). | `[Blocking]` |
| **Lockfile Policy (`.npmrc`)** | `fastify/fastify`, `@fastify/*` plugins, and utilities set `package-lock=false` in `.npmrc`—`package-lock.json` must **not** be committed (`fastify/website` is the exception and tracks `package-lock.json`). | `[Blocking]` |
| **No Stray Generated/Ignored Files** | Reject committed `coverage/`, `.nyc_output/`, `dist/`, `build/`, or `fastify/website` build-time data (`src/content/docs/`, `public/docs/`, `src/data/*.json`). | `[Blocking]` |
| **Semver & Target Branch** | Breaking changes (`feat!:` / major bumps) must target `next` (or be explicitly coordinated for a major release), while bug fixes and non-breaking features target `main` (or backport branches `v4.x` / `5.x`). | `[Blocking]` or `[Question]` |
| **Dependencies & `.npmrc` Security** | New runtime dependencies should be strongly justified (Fastify minimizes overhead and supply-chain surface). `.npmrc` `ignore-scripts=true` and `min-release-age=7` must not be weakened. | `[Blocking]` |

---

## 3. Repository-Specific Technical Review Checklists

Pair with the matching repository skill and verify the checklist for the target codebase:

### A. `fastify/fastify` Core (with `fastify-core`)
- [ ] **V8 Monomorphic Shapes**: Does `Request` (`lib/request.js`) or `Reply` (`lib/reply.js`) initialize all properties in the constructor (`null`/`undefined`), avoiding dynamic property addition/deletion? (`test/same-shape.test.js`)
- [ ] **Hot-Path Discipline**: Are per-request closures, unnecessary `async/await` wrappers, or `Object.assign` allocations avoided in `lib/handle-request.js`, `lib/hooks.js`, `lib/Reply`, and `lib/Request`?
- [ ] **Precompiled Standalone Artifacts**: If `build/build-validation.js` or `build/build-error-serializer.js` changed, was `npm run build:validation` or `npm run build:error-serializer` run and committed (`npm run test:validator:integrity`)?
- [ ] **Error & Warning Catalog**: Are new errors defined in `lib/errors.js` via `createError('FST_ERR_...', ...)` and documented in `docs/Reference/Errors.md` with unit tests in `test/internals/errors.test.js`? Are warnings in `lib/warnings.js` and `docs/Reference/Warnings.md`?
- [ ] **TypeScript Generics (`fastify.d.ts` + `types/*.d.ts`)**: Are type parameters (`RawServer`, `RawRequest`, `RawReply`, `Logger`, `TypeProvider`) propagated and tested in `test/types/*.tst.ts` (`npm run test:types`)?
- [ ] **100% Line Coverage**: Are all new runtime branches covered by `borp` unit tests (`npm run unit` / `npm run coverage:ci-check-coverage`)?
- [ ] **Ecosystem Docs (`docs/Guides/Ecosystem.md`)**: If a plugin was added to `Ecosystem.md`, is it in strict alphabetical order (`npm run lint:ecosystem`)?

### B. `@fastify/*` Plugins & Utilities (with `fastify-plugins`)
- [ ] **Plugin Wrapper & Metadata**: Is the plugin wrapped in `fp(plugin, { fastify: '5.x', name: '@fastify/<name>', decorators: ..., dependencies: ... })`?
- [ ] **CJS / ESM Export Triplet**: For CJS plugins, does `index.js` export `module.exports = fastify<Name>`, `.default = fastify<Name>`, and `.fastify<Name> = fastify<Name>`?
- [ ] **Registration vs. Request Time**: Are schemas compiled, options validated, and decorators registered at **plugin registration time** rather than per-request?
- [ ] **Resource Teardown**: Does any plugin managing connections, timers, or pools register an `onClose` hook to clean up on `fastify.close()`?
- [ ] **Runtime + `.d.ts` + TSTyche Lockstep**:
  - Do `types/index.d.ts` (including `declare module 'fastify'` augmentations) and `types/index.tst.ts` (`tstyche`) match every new or modified runtime option, decorator, and hook?
  - Are both positive and negative (`expect(...).type.not.toBeAssignableTo(...)` / `.toRaiseError()`) type assertions included when constraints change?
- [ ] **100% Test Coverage**: Do unit tests in `test/*.test.js` use `node:test` + `t.assert.*` (or `@matteo.collina/tspl`), prefer `fastify.inject()` over real sockets unless testing network boundaries, and maintain `100%` coverage (`c8 --100` / `borp -C --check-coverage`)?

### C. `fastify/workflows` (with `fastify-workflows`)
- [ ] **Backward Compatibility of `workflow_call`**: Are existing inputs, types, and default values preserved so 60+ caller repos do not break?
- [ ] **40-Character Commit SHA Pinning**: Is every `uses:` action pinned to a full 40-char commit SHA with a `# vX.Y.Z` version comment?
- [ ] **Shell-Injection Hardening**: Are untrusted GitHub contexts (`github.event.inputs.*`, `github.head_ref`, PR titles/bodies) passed via `env:` variables and double-quoted in `run:` blocks instead of interpolated with `${{ ... }}`?
- [ ] **Least-Privilege Permissions**: Does the workflow set `permissions: {}` or `permissions: contents: read` at the top level and scope write permissions to individual jobs?
- [ ] **Schema & Documentation Sync**: Does `action-validator` pass on all changed `.github/workflows/*.yml` files, and are input tables in `README.md` updated in lockstep?

### D. `fastify/website` (with `fastify-website`)
- [ ] **Source-of-Truth Guard**: Does the PR avoid editing or committing git-ignored files (`src/content/docs/`, `public/docs/`, `src/data/*.json`)? If the author wants to change Fastify docs or ecosystem entries, direct them to `fastify/fastify/docs/`.
- [ ] **Base-Aware URLs**: Do all internal root-absolute links and assets use `withBase(...)` from `~/lib/href`?
- [ ] **XSS Sanitization**: Are `set:html` and client-side `innerHTML` sanitized with `DOMPurify.sanitize(...)` (`isomorphic-dompurify`) or `escapeHtml`?
- [ ] **Theme, UI & Accessibility (`frontend-design` & `web-design-guidelines`)**:
  - Do components use semantic tokens from `src/styles/tokens.css` so both dark (default) and `[data-theme="light"]` themes have proper contrast?
  - Do UI changes comply with `.agents/skills/frontend-design/SKILL.md` and `.agents/skills/web-design-guidelines/SKILL.md`?
- [ ] **Build Data Resilience**: Do changes in `scripts/*.mjs` succeed locally even when `GH_TOKEN` is unset?
- [ ] **Biome & Astro Check**: Do `npm run lint` (`biome check`), `npm run check` (`astro check`), and `npm run build:website` pass?
