# `fastify-maintainer` Evaluation Report (Phase 4)

## 1. Executive Summary

Evaluated [`skills/fastify-maintainer/SKILL.md`](../../skills/fastify-maintainer/SKILL.md) across all **9 curated repository cases** ([`cases.json`](cases.json)) at pinned commits from [`evals/repositories.json`](../repositories.json), comparing **With-Skill** vs **Without-Skill (Baseline)** subagent runs in isolated repository checkouts (`evals/results/cases/CASE-001` through `CASE-009`).

### Aggregate Quantitative Comparison

| Metric | With `fastify-maintainer` Skill | Without Skill (Baseline) |
| --- | --- | --- |
| **Workflow Selection Accuracy** (`review` vs `implementation` vs `scope-guard`) | **9 / 9 (100%)** | 8 / 9 (89% — proceeded with patch suggestions on unmanned plugin `CASE-009`) |
| **Read-Only Discipline on Reviews (`CON-003`)** | **4 / 4 (100%)** | 4 / 4 (100%) |
| **8-Area Applicability Matrix & Non-Applicable Reasons (`REQ-003`)** | **9 / 9 (100%)** | 1 / 9 (11%) |
| **Explicit Integration-Test Boundary Decision (`REQ-006`)** | **9 / 9 (100%)** | 1 / 9 (11%) |
| **Pre-Fix Failure Reproduction Before Patch (`REQ-005`)** | **2 / 2 (100%)** (`CASE-003`, `CASE-005`) | 1 / 2 (50% — patched `CASE-005` before running pre-fix `tstyche`) |
| **Positive + Negative TSTyche Assertions (`TASK-008`)** | **2 / 2 (100%)** (`CASE-004`, `CASE-005`) | 1 / 2 (50% — omitted negative matcher on `CASE-004`) |
| **Maintainer PR Title (`action-pr-title`) & Human-PR Policy (`CON-004`)** | **5 / 5 (100%)** (all implementation cases) | 0 / 5 (0%) |
| **Clean Control — Zero Invented Blockers (`CASE-002`)** | **Pass** (`No actionable defects found.`) | **Pass** |

---

## 2. Case-by-Case Breakdown (`CASE-001` – `CASE-009`)

| Case | Repository (`pinned_sha`) | Task Type | With-Skill Outcome | Baseline Outcome | Key Differentiation |
| --- | --- | --- | --- | --- | --- |
| **CASE-001** | `fastify/fastify` (`9e6d963f09`) | Review (Runtime + `.d.ts` regression) | Found both blockers with exact file/line evidence (`lib/hooks.js:436-445` mixing `async` and `done()`, `lib/route.js:697-698` missing `fastify.d.ts` + `test/types/` TSTyche test), included 8-area matrix and `fastify.inject()` rationale. | Found both defects, but omitted the 8-area matrix, integration-test boundary rationale, and `test:validator:integrity` check. | Structured 8-area coverage and explicit integration boundary decision. |
| **CASE-002** | `fastify/fastify-sensible` (`e3908a1e89`) | Review (Clean PR control) | Ran `npm run lint`, `npm run test:unit` (`borp`, 168/168, 100% cov), `npm run test:typescript` (`tstyche`, 108/108); explicitly reported **`No actionable defects found.`** | Ran tests and approved PR without structured area breakdown or integration rationale. | Zero false positives while maintaining complete verification evidence. |
| **CASE-003** | `fastify/fastify-plugin` (`881fb922ef`) | Implement (Bug fix + regression test) | Reproduced failure first via `node --test test/test.js`, fixed `index.js` (`options.name.trim() === ''`), added regression test, passed `npm run test:unit` (48/48, 100% cov), `npm run test:typescript` (39/39), and `npm run lint`. | Implemented fix and passed tests, but omitted pre-fix failure log, integration rationale, and `action-pr-title` / human-PR policy summary. | Reproduce-first discipline and maintainer PR readiness. |
| **CASE-004** | `fastify/fastify-cookie` (`576e0ddcac`) | Implement (Feature across JS, `.d.ts`, TSTyche, docs) | Verified `partitioned` option across `index.js`, `types/index.d.ts`, `test/cookie.test.js` (`fastify.inject()`), `README.md`, and added both positive & negative TSTyche assertions in `types/index.tst.ts` (48/48 passing). | Verified existing test coverage (46/46 TSTyche assertions) without adding a negative TSTyche assertion or recording integration boundary rationale. | Stronger positive + negative TSTyche coverage (`expect(...).type.not.toBeCallableWith(...)`). |
| **CASE-005** | `fastify/fastify-type-provider-typebox` (`374aeb2d13`) | Implement (Type-only regression) | Added positive & negative TSTyche assertions in `types/index.tst.mts`, confirmed pre-fix failure (`npm run test:types` exit code `1`), fixed `serializer` in `index.ts` without touching runtime code, passed `npm test` (`build` + `attw` + 38/38 node tests + 22/22 `tstyche`) and `npm run lint`. | Fixed `index.ts` and added positive TSTyche test, but did not record pre-fix `tstyche` failure or negative type assertion. | Pre-fix type-test failure verification and zero unnecessary runtime edits. |
| **CASE-006** | `fastify/fastify-multipart` (`dc4253b8cd`) | Implement (Socket/stream integration boundary) | Explicitly justified why `fastify.inject()` is insufficient for TCP socket abort/FD leak verification and required `await fastify.listen({ port: 0 })` + `t.after(() => fastify.close())` + `client.destroy()`. | Explained the socket abort test setup but lacked the 8-area applicability table and maintainer PR summary. | Explicit transport-boundary rationale (`REQ-006`). |
| **CASE-007** | `fastify/workflows` (`b1a11bce71`) | Review (CI-only diff) | Discovered `"scripts": {}` in `package.json`, marked JS/TS/runtime/type/integration tests as **Not Applicable**, and audited `plugins-ci.yml` matrix, `workflow_call` inputs, and SHA-pinned actions. | Audited the workflow well, but did not include the full 8-area applicability matrix. | Clean handling of repositories without JS/TS test suites. |
| **CASE-008** | `fastify/website` (`9ae2326173`) | Implement (Docs-only with Biome) | Discovered `biome check` (`npm run lint` in `package.json`), marked Type/Integration tests as **Not Applicable**, updated docs without assuming `neostandard` or `borp`, and honestly disclosed missing local `node_modules/.bin/biome`. | Identified Biome and docs pipeline, but lacked the 8-area matrix and PR readiness block. | Accurate non-standard linter discovery (`biome check`) and honest local environment reporting. |
| **CASE-009** | `fastify/fastify-opensearch` (`cd63997a0a`) | Scope Guard (Unmanned plugin) | Halted immediately during Step 1 (`repository-discovery.md`), citing that `fastify-opensearch` is absent from the 60 Core Plugins on `https://fastify.dev/ecosystem/` (**unmanned**), leaving tracked files untouched. | Noted `fastify-opensearch` is not on the ecosystem page, but still performed a code review and proposed feature patches. | Strict enforcement of the maintainer scope rule (`REQ-008`). |

---

## 3. Trigger & Description Evaluation (`TASK-014`)

Evaluated [`evals/fastify-maintainer/triggers.json`](triggers.json) (15 prompts) against the `SKILL.md` frontmatter `description`:

- **Intended Activations (10/10 matched):**
  - Core (`TRIG-001` `fastify/fastify`), official ecosystem plugins (`TRIG-002` `@fastify/cookie`), non-`fastify`-prefixed ecosystem plugins (`TRIG-003` `middie`, `TRIG-004` `under-pressure`, `TRIG-006` `point-of-view`), non-`fastify`-prefixed shared libraries (`TRIG-005` `avvio`, `TRIG-010` `fast-uri`), type providers (`TRIG-007` `fastify-type-provider-typebox`), CI workflows (`TRIG-008` `fastify/workflows`), and website/docs (`TRIG-009` `fastify/website`).
- **Near-Miss Exclusions (3/3 excluded):**
  - End-user SaaS application development (`TRIG-011`), unrelated Express.js PR review (`TRIG-012`), and npm release publishing / org admin permissions (`TRIG-013`) are explicitly excluded by the `SKILL.md` description: *"Do not use for building end-user Fastify applications, publishing releases, or repository administration."*
- **Scope-Guard Triggers (2/2 guarded):**
  - Unmanned plugin (`TRIG-014` `fastify/fastify-opensearch`) and >2-year inactive repository (`TRIG-015` `fastify/collaborator-discussions`) activate discovery Step 1 and immediately halt with an explicit scope warning.
