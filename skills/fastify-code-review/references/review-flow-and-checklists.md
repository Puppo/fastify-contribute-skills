# Fastify PR Review Flow & Technical Checklists

Detailed reference for inspecting diffs, verifying Fastify organization governance, and running repository-specific technical checks during a pull request review.

## 1. Diff & Metadata Inspection Commands

Always inspect the full context of the change before writing findings:

### When Reviewing a GitHub PR (`gh` CLI available)

```bash
# Select the target repository explicitly, including for fork PRs.
gh pr view <number> --repo <owner/repo> --json url,title,body,baseRefName,baseRefOid,headRefName,headRefOid,headRepository,headRepositoryOwner,labels,author,files
gh pr diff <number> --repo <owner/repo>
gh pr checks <number> --repo <owner/repo>
```

Record the base repository from the PR URL and both commit SHAs. Read surrounding source at the head SHA; compare policies against the base when they change. Recheck PR metadata before reporting: if either SHA moved, refresh the review or explicitly limit it to the recorded revisions. Record the revision shown by each remote CI run (head or synthetic merge commit); do not equate remote CI with local execution.

### When Reviewing a Local Branch or Commit Range

```bash
# Inspect the checkout without changing the user's files or index.
git status --short
git rev-parse HEAD

# Replace these placeholders with the confirmed base and head commit SHAs.
git log --oneline <base-sha>..<head-sha>
git diff --stat <base-sha>...<head-sha>
git diff <base-sha>...<head-sha>
```

Use the actual PR base or user-requested base, not a hardcoded `origin/main`. If no base can be established from the request or repository metadata, ask. For an explicitly requested working-tree review, include staged/unstaged changes in scope and label results as working-tree validation rather than validation of a clean commit.

### Revision and Isolation Requirements for Local Checks

1. Compare local `HEAD` with the recorded head SHA and inspect staged, unstaged, and untracked files that could affect execution. A matching SHA with local edits is not a clean test of the PR.
2. If the revision differs or local files could contaminate results, create a temporary detached checkout outside the user's working tree. Fetch the PR head from the identified repository if needed and verify its SHA. For an explicitly requested working-tree review, include the scoped edits in a temporary copy when isolation is needed and record the base SHA plus patch scope. Do not switch, reset, stash, or clean the user's checkout.
3. Read the actual script bodies, lifecycle hooks, and workflow commands before running checks. Regeneration checks such as core's `test:validator:integrity` rewrite tracked files and must run in the temporary checkout, even when the user's checkout already matches the PR.
4. Run only relevant checks with available prerequisites. Record the starting revision, exact commands, exit codes, and resulting generated-file differences. Do not stage regenerated output to make an integrity check pass. If a check unexpectedly changes the user's files, stop and report it rather than resetting them.
5. If the head cannot be obtained or dependencies/services are unavailable, report `Not run` and the reason. Tests on a different revision cannot substitute. Remove only temporary resources created for this review when finished.

- **Read surrounding context**: Verify affected functions, declarations, and tests at the reviewed revision. Never judge a hunk in isolation.
- **Evidence sources**: [GitHub CLI PR metadata fields](https://cli.github.com/manual/gh_pr_view) and [PR checks](https://cli.github.com/manual/gh_pr_checks).

---

## 2. Repository-Specific Governance & PR Hygiene

Read contribution rules, `package.json`, `.npmrc`, ignore rules, tracked files, and workflows in the target checkout. The technical references describe common patterns; confirmed local policy determines applicability. Use `[Question]` for material uncertainty, not an assumed blocking rule.

| Check | Evidence and applicability |
|-------|----------------------------|
| **PR title** | Read the configured title check, including allowed types and backport prefixes. Flag a confirmed violation rather than enforcing a fixed organization-wide list. |
| **Version changes** | Apply the contribution policy to ordinary PRs. Allow version bumps in documented maintainer/automation release PRs, such as the release PR used by `fastify/workflows`; verify they follow that release process. |
| **Lockfiles** | Check existing tracked lockfiles, package-manager configuration, and contribution rules. Preserve a tracked website lockfile; do not infer a universal ban from package name or a setting alone. |
| **Generated files** | Flag unintended generated output based on ignore/build rules and tracked history. Core's `build/` contains source scripts, and `lib/config-validator.js` and `lib/error-serializer.js` are committed generated artifacts. Website docs/data and disposable coverage output have different policies. |
| **Semver and target branch** | Determine supported version lines and release coordination from the target repository. Do not assume every major change targets `next` or every fix targets `main`. |
| **Dependencies and install policy** | Assess new dependencies and changes to install protections against documented requirements and concrete risks. Report confirmed security regressions; ask when an exception's rationale is missing. |
| **Coverage** | Read configured metrics and thresholds in package scripts, runner config, and CI. Report unit results and coverage checks separately; do not infer branch/function coverage from a line-only gate. |

### RFC Currency and Adherence

Apply when the PR or task links or includes protocol standards or project design RFCs:

1. **Read the source and discussion.** Read the RFC itself, included copies, and relevant PR discussion or linked decisions; a PR summary is insufficient. For GitHub PRs, inspect comments as well as the body.
2. **Check currency at the authoritative source.** For protocol RFCs, check status, updates, obsoleting documents, and relevant errata. For project RFCs, check revision history, superseding proposals, acceptance status, and agreed amendments.
3. **Establish the applicable revision.** Record its link/version and why it applies to this change. A newer draft does not automatically supersede an accepted proposal or a deliberately targeted protocol version. Resolve material ambiguity as a question before claiming compliance.
4. **Check the relevant facets.** Compare affected runtime behavior, public types, tests, and documentation against applicable requirements, including normative strength, conditions, exceptions, optional behavior, and compatibility considerations. Check both normal and failure paths where required; do not treat optional behavior as mandatory.
5. **Report with evidence.** Cite the applicable RFC section alongside the changed `file:line` and explain any demonstrated violation. Use `[Question]` for unresolved interpretation or target-version ambiguity. If a source is inaccessible, disclose the limitation and do not claim RFC compliance.

---

## 3. Repository-Specific Technical Review Checklists

Pair with the matching repository skill and apply only checks relevant to the changed behavior. Confirm commands and paths in the target checkout before executing examples below.

### A. `fastify/fastify` Core (with `fastify-core`)
- [ ] **Keep Core Minimal**: Anything that can be implemented as a plugin should be a plugin. Do existing extension points support the proposed behavior? Require a concrete reason for core placement; use `[Question]` when feasibility or rationale is unclear.
- [ ] **V8 Monomorphic Shapes**: Does `Request` (`lib/request.js`) or `Reply` (`lib/reply.js`) initialize all properties in the constructor (`null`/`undefined`), avoiding dynamic property addition/deletion? (`test/same-shape.test.js`)
- [ ] **Hot-Path Discipline**: Are per-request closures, unnecessary `async/await` wrappers, or `Object.assign` allocations avoided in `lib/handle-request.js`, `lib/hooks.js`, `lib/Reply`, and `lib/Request`?
- [ ] **Precompiled Standalone Artifacts**: If `build/build-validation.js` or `build/build-error-serializer.js` changed, do committed generated files match their sources? Confirm `build:validation` and `test:validator:integrity` in `package.json`; run the integrity script only in the isolated checkout described above.
- [ ] **Error & Warning Catalog**: Are new errors defined in `lib/errors.js` via `createError('FST_ERR_...', ...)` and documented in `docs/Reference/Errors.md` with unit tests in `test/internals/errors.test.js`? Are warnings in `lib/warnings.js` and `docs/Reference/Warnings.md`?
- [ ] **TypeScript Generics (`fastify.d.ts` + `types/*.d.ts`)**: Are type parameters (`RawServer`, `RawRequest`, `RawReply`, `Logger`, `TypeProvider`) propagated and tested in `test/types/*.tst.ts` (`npm run test:types`)?
- [ ] **Runtime and Coverage**: Do relevant regression tests pass, and does the configured coverage gate pass separately? Confirm scripts such as `unit` and `coverage:ci-check-coverage` in the checkout before running them.
- [ ] **Ecosystem Docs (`docs/Guides/Ecosystem.md`)**: If a plugin was added to `Ecosystem.md`, does it meet the ordering rule? Inspect `.github/workflows/lint-ecosystem-order.yml` and execute its actual validation command with the same prerequisites; do not invent a package script.

### B. Fastify Plugins & Utilities (with `fastify-plugins`)

- [ ] **Package classification**: Inspect exports and implementation. Registered plugins, type providers, and standalone utilities have different contracts; apply only the relevant checks below.
- [ ] **Plugin scope and metadata**: For a registered plugin that exposes behavior outside its scope, verify appropriate `fastify-plugin` wrapping and metadata matching supported Fastify versions. Preserve intentionally encapsulated route plugins. Utilities do not require wrapping or registration metadata.
- [ ] **Exports**: Preserve the package's supported CJS/ESM entrypoints and named exports. Check the export triplet where that plugin provides it; do not impose it on utilities or alternative module layouts.
- [ ] **Lifecycle**: For plugins, validate registration-time setup, hooks, decorators, and resource teardown. For utilities, test their own API, errors, and resource ownership without introducing Fastify lifecycle machinery.
- [ ] **Public types and tests**: Update declarations and relevant type tests when the public contract changes, using the checkout's paths and runner. For TSTyche, test invalid calls with `.not.toBeCallableWith(...)`; do not recommend `.toRaiseError()` for new tests.
- [ ] **Runtime and coverage**: Test changed behavior and regression cases through the appropriate API. Use the repository's runner and coverage thresholds; `inject()` applies to request behavior, not all utility tests.

### C. `fastify/workflows` (with `fastify-workflows`)
- [ ] **Backward Compatibility of `workflow_call`**: Are existing inputs, types, defaults, and caller permissions preserved? Inspect the representative caller scenarios in [the workflow validation guide](../../fastify-workflows/references/validation-and-security.md).
- [ ] **40-Character Commit SHA Pinning**: Are external actions pinned to full 40-char commit SHAs with version comments? Relative local workflow references remain paths without SHAs.
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
- [ ] **Browser Behavior**: For UI/routing changes, exercise affected scenarios in [the production preview checklist](../../fastify-website/references/validation-and-workflows.md): version-scoped search, keyboard/focus, themes, version switching, Markdown routes, and non-root base paths. Report browser execution separately from the build.
