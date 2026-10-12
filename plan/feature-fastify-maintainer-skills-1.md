---
goal: Build portable maintainer skills for reviewing and changing repositories across the Fastify organization
version: 1
date_created: 2026-10-09
last_updated: 2026-10-11
owner: Fastify maintainers
status: In Progress
tags: [skills, fastify, review, implementation, testing]
---

# Introduction

![Status: In Progress](https://img.shields.io/badge/status-In%20Progress-yellow)

Build a repository-aware maintainer skill that helps review pull requests, implement features, and fix bugs across actively maintained repositories in the [fastify](https://github.com/fastify) GitHub organization. Portable `SKILL.md` support for multiple coding agents is a confirmed requirement.

The in-scope repositories within `https://github.com/fastify` are strictly bounded by two rules:
1. **Official Plugins**: For Fastify plugins, **only** include core plugins listed on [https://fastify.dev/ecosystem/](https://fastify.dev/ecosystem/). Any plugin repository in the `fastify` organization not listed on the ecosystem page is considered unmanned and out of scope.
2. **Active Maintenance (2-Year Commit Cutoff)**: Across both ecosystem plugins and non-plugin organization repositories (core `fastify/fastify`, core utilities/plumbing like `avvio`, `fastify-plugin`, `light-my-request`, `fast-json-stringify`, `@fastify/error`, shared `workflows`, `website`, etc.), **exclude any archived repository or any repository with no commits in the last 2 years** relative to the inventory date.

Start with one installable `fastify-maintainer` skill containing review and implementation workflows. Keep detailed guidance in focused references loaded only when relevant. Split into independently installable skills later only if evaluation demonstrates that separate entry points improve task selection or outcomes.

Phases 1–3 (repository inventory, evaluation case design, entry-point skill, review/implementation workflows, and all 8 code-area references) are implemented in this workspace; Phases 4–5 (isolated behavioral evaluation and multi-agent pilot) remain next.

## 1. Requirements & Constraints

- **REQ-001**: Support PR review, new features, and bug fixes in the checked-out repository and target branch.
- **REQ-002**: Discover repository instructions, contribution policy, package/workspace boundaries, public entry points, source layout, generated files, supported versions, package manager, tests, CI, and documentation before choosing changes or commands. Record evidence paths and unresolved gaps.
- **REQ-003**: Cover every area in the matrix below. Mark irrelevant areas with a reason; do not invent missing test suites or impose new tooling.
- **REQ-004**: Review output must identify the reviewed base/head, actionable findings with file/line evidence, trigger and impact, severity, validation performed, and remaining uncertainty. Separate defects from optional suggestions; explicitly report when no actionable findings were found.
- **REQ-005**: Implementation output must explain the behavior changed, regression or acceptance coverage, compatibility implications, documentation changes, commands executed and results, and checks that could not run.
- **REQ-006**: Determine integration-test needs from the changed boundary. In-process tests (`fastify.inject()`) can cover component integration; require real sockets (`listen({ port: 0 })`), processes, services, or installed-package consumers only when those boundaries affect the behavior being verified.
- **REQ-007**: Use the target checkout's policies and tooling as the source of truth. Retrieve version-appropriate official documentation when an API or tool detail requires verification; do not substitute latest documentation for the target branch's contract.
- **REQ-008**: Filter the `https://github.com/fastify` repository inventory using two mandatory criteria: (a) plugin repositories must appear in the official Core Plugins list on `https://fastify.dev/ecosystem/` (unlisted plugin repos are unmanned and excluded), and (b) all repositories (plugins and non-plugin org repos) must have at least one commit within the last 2 years and not be archived.
- **REQ-009**: Distribute portable `SKILL.md` instructions with relative references for multiple coding agents. Validate installation, reference loading, and representative review/implementation behavior in at least two agent environments before claiming tested multi-agent support; record agents and versions used.
- **CON-001**: Keep the base skill portable Markdown with relative references. Do not require a specific agent, MCP server, GitHub connector, or sibling skill. Support review from a local diff when remote tooling is unavailable.
- **CON-002**: Do not hard-code Node.js or TypeScript versions, package-manager commands, test runners, branch names, coverage thresholds, or core-only paths across repositories.
- **CON-003**: For review-only requests, report findings without changing tracked files. Implementation requests permit scoped edits and local validation. Publishing, releases, and repository administration are outside this skill's initial scope.
- **CON-004**: Inspect contribution policies governing AI assistance and PR submission. Fastify core currently requires human-opened PRs; produce a reviewable patch and PR text consistent with the target repository's policy.
- **GUD-001**: Prefer existing abstractions and test patterns. Add scripts only after evaluations identify a repeated task that benefits from deterministic automation.

| Area | Guidance to include | Evidence expected when applicable |
| --- | --- | --- |
| Codebase | Trace affected entry points, callers, architecture, compatibility contracts, generated sources, and package boundaries. Verify Fastify encapsulation (`fastify-plugin` / `Symbol.for('skip-override')`), decorator/hook inheritance, lifecycle ordering, and schema compilation where present. | Impact map, target branch, local conventions, compatibility assessment. |
| JavaScript | Match local module format (CJS/ESM) and style (`neostandard` / `@fastify/eslint-config`); inspect hot-path performance (registration-time vs request-time allocations, monomorphic shapes, avoiding mixing `async` and `done` callbacks), `@fastify/error` (`FST_ERR_*`) codes, `fast-warning` (`FSTDEP*`) deprecations, security boundaries (prototype pollution, ReDoS), and resource cleanup. | Runtime behavior checks; benchmarks for justified performance changes. |
| TypeScript | Keep runtime JS and manual `.d.ts` declarations (`index.d.ts`, `types/*.d.ts`) or TS sources strictly synchronized; inspect Fastify module augmentation (`declare module 'fastify'`), generics (`RawServer`, `TypeProvider`, `SchemaCompiler`), inference, overloads, and export resolution. | Declaration/source alignment and consumer compatibility evidence. |
| Runtime tests | Reproduce bugs and cover new behavior, errors, and edge cases with the repository's runner (`borp` / `node:test` + `@matteo.collina/tspl` with `await t.completed`, or `tap`). Isolate state and close resources (`t.after(() => fastify.close())`). | A regression fails for the intended reason before the fix and passes afterward, or a documented reason reproduction was unavailable. |
| Type tests | Verify accepted and rejected usage, inference, overload selection, and regression boundaries using the repository's type-test runner (**TSTyche** or `tsd` / `tsc`). When `tstyche` is used or being adopted, follow [TSTyche Agent Skills](https://tstyche.org/guides/agent-skills) (`tstyche/tstyche-skills`: `expect(x).type.toBe<...>()`, ability matchers like `.toBeCallableWith`, and `rejectAnyType`/`rejectNeverType`); when `tsd` is used, check `expectType`, `expectError`, `expectAssignable`, `expectDeprecated` in `test/types/*.test-d.ts`. | Meaningful positive/negative assertions, including a pre-fix failure for type regressions where feasible. |
| Integration tests | Default to in-process `fastify.inject()` (`light-my-request`); require real ephemeral sockets (`listen({ port: 0 })`), child processes, external services (Redis/Postgres/Kafka), or package-export consumers only when verifying transport, stream abort, multipart boundary, or external driver behavior. | Explicit required/not-required decision with rationale; fixtures and cleanup when required. |
| CI | Inspect workflow triggers, matrices, reusable workflows (`fastify/workflows`), linter/typecheck steps, caches, permissions, and service containers relevant to the diff. | Applicable jobs still run, version coverage is preserved, and unavailable matrix checks are identified. |
| Docs | Update public options, defaults, examples, error/deprecation codes, compatibility tables, and migration notes when behavior changes. Follow the repository's release-note policy. | Documentation agrees with runtime/types; existing documentation checks pass where available. |

## 2. Implementation Steps

### Implementation Phase 1: Establish filtered repository inventory and evaluation cases

- **GOAL-001**: Define a source-backed repository profile restricted to `https://fastify.dev/ecosystem/` core plugins and active non-plugin org repositories (commits within the last 2 years), plus a representative evaluation set.

| Task | Description | Completed | Date |
| --- | --- | --- | --- |
| TASK-001 | Create `evals/repositories.json`. Cross-reference `https://github.com/fastify` repositories against `https://fastify.dev/ecosystem/` and last-commit timestamps: (1) include official plugins only if listed on `https://fastify.dev/ecosystem/` AND committed to within the last 2 years; (2) include non-plugin org repositories (core, plumbing/utilities, standalone parsers/serializers, type providers, workflows, docs/website) only if not archived AND committed to within the last 2 years; (3) exclude unmanned plugins and repos inactive for >2 years. Select representative repositories per active category with pinned URLs and commit SHAs. | Yes | 2026-10-11 |
| TASK-002 | After TASK-001, create `skills/fastify-maintainer/references/repository-discovery.md`. Define discovery evidence for instructions, unmanned/inactive repository detection, workspaces, runtime/compiler support, module format, test runner (`borp`/`node:test`/`tspl` vs `tap`), type tests (`tstyche` vs `tsd` vs `tsc`), linter (`neostandard`/`standard`), CI (`fastify/workflows`), and docs. Handle absent `package.json`, absent type tests, and conflicting evidence without guessing commands. | Yes | 2026-10-11 |
| TASK-003 | After TASK-001, create `evals/fastify-maintainer/cases.json` with realistic review, feature, bug-fix, type-only, CI-only, docs-only, integration-required, and integration-unnecessary tasks drawn from the active in-scope repositories. Record expected observable outcomes separately from task prompts. | Yes | 2026-10-11 |

Completion: every requested code area maps to discovery evidence and at least one evaluation case; unmanned plugins (not on `fastify.dev/ecosystem`) and repositories without commits in the last 2 years are excluded; representative active repositories are selected across categories.

### Implementation Phase 2: Build the entry point and workflows

Dependencies: Phase 1.

- **GOAL-002**: Produce one usable skill with clear task selection, scope verification (warning maintainers if a target plugin repo is unmanned or inactive >2 years), and distinct review/implementation behavior.

| Task | Description | Completed | Date |
| --- | --- | --- | --- |
| TASK-004 | Create `skills/fastify-maintainer/SKILL.md` with name/description frontmatter, maintainer-focused activation, repository discovery (including checking whether a plugin is in the `fastify.dev/ecosystem` list and active within 2 years), task selection, applicability assessment, and links to conditional references. Keep generic Fastify application development outside automatic activation. | Yes | 2026-10-11 |
| TASK-005 | Create `skills/fastify-maintainer/references/review.md`: establish the actual PR base/head or local diff, trace changed contracts and consumers, inspect applicable areas, validate suspected issues, and emit evidence-backed findings without editing tracked files. If the comparison cannot be determined, state the missing input. | Yes | 2026-10-11 |
| TASK-006 | Create `skills/fastify-maintainer/references/implementation.md`: establish expected behavior, trace affected code, reproduce bugs, add focused coverage, make scoped changes, update applicable types/docs, execute repository-required checks, and summarize the resulting patch. | Yes | 2026-10-11 |

Completion: walkthroughs for one review, one feature, and one bug fix select the correct workflow and produce the required outputs without assuming Fastify core's tooling.

### Implementation Phase 3: Add focused code-area guidance

Dependencies: Phase 2. Tasks in this phase may be completed independently before routing verification.

- **GOAL-003**: Cover all eight areas through Fastify-specific maintainer references without duplicating generic language tutorials.

| Task | Description | Completed | Date |
| --- | --- | --- | --- |
| TASK-007 | Create `references/codebase-javascript.md` inside the skill. Cover architecture/compatibility tracing, Fastify lifecycle and encapsulation (`fastify-plugin`, `Symbol.for('skip-override')`), callback/Promise completion rules, `@fastify/error` (`FST_ERR_*`) and `fast-warning` (`FSTDEP*`), hot-path performance patterns, security checks (prototype pollution, ReDoS), module exports, and justified benchmark validation. | Yes | 2026-10-11 |
| TASK-008 | Create `references/typescript.md`. Cover runtime JS and manual `.d.ts` synchronization, Fastify declaration merging (`declare module 'fastify'`), generic propagation (`TypeProvider`, `SchemaCompiler`, `RawServer`), overloads, export resolution, and compiler compatibility. For type tests, integrate guidance from [TSTyche Agent Skills](https://tstyche.org/guides/agent-skills) (`tstyche/tstyche-skills`: `tstyche-type-tests`, `tstyche-project-setup`, `tstyche-migrate-from-tsd`)—delegating to installed TSTyche skills when present while including built-in rules for both `tstyche` (`.tst.ts` / `.test-d.ts`, `expect(x).type.*` relation & ability matchers) and `tsd` (`test/types/*.test-d.ts`). | Yes | 2026-10-11 |
| TASK-009 | Create `references/testing.md`. Cover regression and feature tests across `borp`/`node:test` + `@matteo.collina/tspl` (`await t.completed`) and `tap`, test isolation, resource cleanup (`fastify.close()`), `fastify.inject()` versus real socket/service integration decision criteria, and targeted versus full repository checks. | Yes | 2026-10-11 |
| TASK-010 | Create `references/ci-docs.md`. Cover `fastify/workflows` reusable workflows and relevant matrices, `neostandard` / `@fastify/eslint-config` checks, local/remote validation limits, documentation consistency (options, defaults, error/deprecation tables), and repository-specific contribution/release policy. | Yes | 2026-10-11 |
| TASK-011 | After TASK-007 through TASK-010, verify links from `SKILL.md` and both workflows. Each area must have an explicit load condition; routine tasks must not load every reference automatically. | Yes | 2026-10-11 |

Completion: the coverage matrix has no unassigned area, and each reference has a tested routing condition.

### Implementation Phase 4: Evaluate and refine

Dependencies: Phase 3.

- **GOAL-004**: Demonstrate useful behavior across active in-scope repository categories and detect overgeneralization.

| Task | Description | Completed | Date |
| --- | --- | --- | --- |
| TASK-012 | Validate skill metadata, reference resolution, and absence of unfinished scaffolding. Run the cases from TASK-003 in isolated copies at pinned revisions, both with and without the skill under equivalent conditions. Store patch/output, command results, and observed outcomes under `evals/results/`; exclude generated results from distributable skill content. | No | — |
| TASK-013 | Compare outputs for defect detection, false positives, fix correctness, regression strength, applicability decisions, scope control, and verification honesty. Present raw outputs and a concise report to maintainers for feedback; revise only for demonstrated weaknesses. | No | — |
| TASK-014 | Add `evals/fastify-maintainer/triggers.json` covering intended tasks across active Fastify repository types and near misses such as building a new Fastify application, working on unmanned/archived plugins, reviewing unrelated JavaScript, and performing releases. Include active in-scope repositories whose names do not contain Fastify (e.g., `avvio`, `middie`, `under-pressure`, `point-of-view`). Recheck description selection after revisions. | No | — |

Completion: all required outcomes in the curated cases pass; known review defects are found; clean controls produce no invented blocking findings; no invented commands or false test-pass claims occur. Record unresolved cases instead of claiming universal validation.

### Implementation Phase 5: Package and pilot

Dependencies: Phase 4.

- **GOAL-005**: Make the skill independently installable and maintainable across target agents and active in-scope repositories.

| Task | Description | Completed | Date |
| --- | --- | --- | --- |
| TASK-015 | Test isolated installation, reference loading, and representative review/implementation tasks in at least two available coding-agent environments. Update the repository `README.md` with purpose, scope rules (`fastify.dev/ecosystem` plugins + 2-year commit activity), companion skill recommendation (`npx skills add tstyche/tstyche-skills` from `https://tstyche.org/guides/agent-skills`), layout, tested agent versions and installation instructions, invocation examples, contribution workflow, and validation limits. Keep optional agent-specific metadata separate from the portable instructions; record unavailable environments as unvalidated. | No | — |
| TASK-016 | Pilot on maintainer-selected real tasks spanning core (`fastify/fastify`), an official ecosystem plugin, a shared core library/tool, a type-provider repository, and a docs/workflow repository within the active Fastify set. Record feedback, agent environments, and tested revisions in `evals/pilot.md`; rerun affected cases after fixes. | No | — |

Completion: isolated installations in at least two agent environments resolve every reference and complete representative review/implementation tasks without this repository's evaluation files or globally installed companion skills. The pilot covers the selected active Fastify repository categories; limitations and excluded unmanned/inactive repositories are documented.

## 3. Alternatives

- **ALT-001**: Eight separate skills, one per code area. Defer because most maintainer tasks cross multiple areas, making activation and consistent discovery harder.
- **ALT-002**: Separate `fastify-review` and `fastify-implement` packages. Reconsider if maintainers need independent installation or evaluations show ambiguous activation. Any split must preserve self-contained installations and avoid duplicated, drifting guidance.
- **ALT-003**: One large instruction file containing every checklist. Avoid because narrow tasks would load unrelated guidance.
- **ALT-004**: Hard-coded per-repository profiles as runtime requirements. Prefer live discovery; use pinned profiles only as evaluation evidence because repositories evolve independently.

## 4. Dependencies

- **DEP-001**: Read access to selected repositories, relevant contribution policies, and maintainer-selected examples. PR comments are optional evidence; a remote connector is not a runtime requirement.
- **DEP-002**: Each evaluation repository's own runtime, package manager, and test prerequisites. Record unavailable services or matrix environments as validation gaps.
- **DEP-003**: Maintainer feedback for evaluation quality and pilot acceptance. Planning and the initial draft can proceed using the assumptions below.
- **DEP-004**: Access to at least two coding-agent environments for portability validation. Portable packaging can be drafted before these environments are available.

## 5. Files

- **FILE-001**: `skills/fastify-maintainer/SKILL.md` — installable entry point.
- **FILE-002**: `skills/fastify-maintainer/references/` — discovery, review, implementation, codebase/JavaScript, TypeScript/type tests, testing/integration, and CI/docs guidance.
- **FILE-003**: `evals/repositories.json` and `evals/fastify-maintainer/` — pinned repository sample, task cases, expected outcomes, and activation cases.
- **FILE-004**: `evals/results/` and `evals/pilot.md` — isolated evaluation outputs and pilot evidence; generated results excluded from distribution.
- **FILE-005**: `README.md` — repository-level installation, usage, contribution, and support documentation.
- **FILE-006**: `plan/feature-fastify-maintainer-skills-1.md` — this plan and future progress tracking.

## 6. Testing

- **TEST-001**: Review a known runtime regression and a clean change; require supported findings for the former and no invented blockers for the latter.
- **TEST-002**: Fix a runtime bug; require an observable reproduction, a focused patch, and applicable passing checks.
- **TEST-003**: Add a public option; require consistent runtime behavior, declarations where present, applicable runtime/type tests, and docs.
- **TEST-004**: Fix a type inference regression; require meaningful type assertions and no unjustified runtime edits.
- **TEST-005**: Exercise an integration-sensitive change and a purely local change; require correct boundary selection and explicit integration rationale in both.
- **TEST-006**: Review CI-only and docs-only diffs; require applicable validation without introducing unrelated runtime changes or test infrastructure.
- **TEST-007**: Run against a repository with different script names/module conventions and one with no type-test suite; require locally evidenced commands and accurate not-applicable decisions.
- **TEST-008**: Exercise missing dependencies, ambiguous base refs, unavailable services, and skipped CI matrices; require honest limitations without fabricated success.
- **TEST-009**: Test a maintenance branch, workspace/multi-package layout, and docs/non-Node repository; require correct version/package boundaries and no blanket JavaScript assumptions.
- **TEST-010**: Validate metadata and installation portability, reference routing, and intended versus near-miss activation. Structural validation alone does not satisfy behavioral acceptance.
- **TEST-011**: Exercise different repository categories within Fastify, including repository-specific contribution policies. Require locally evidenced conventions and correct activation even without Fastify in the repository name.
- **TEST-012**: Install the same portable skill in at least two coding-agent environments and run representative review/implementation cases. Require resolvable relative references, correct workflow selection, and no dependency on another agent's tools or global skills.

## 7. Risks & Assumptions

- **RISK-001**: A small repository sample can overfit to core and popular plugins. Mitigate with category coverage, differing tools, and maintenance-branch cases; explicitly track untested categories.
- **RISK-002**: Shared guidance can become stale. Keep commands and supported versions dynamically discovered and record source revisions in evaluations.
- **RISK-003**: Excessive checklist use can create noisy reviews and unnecessary tests. Tie each check to the diff and evaluate clean controls and narrow tasks.
- **RISK-004**: Agent-specific installation or tools can undermine portability. Test an isolated portable installation before adding optional integrations.
- **ASSUMPTION-001**: This work currently produces a plan; skill implementation, evaluation execution, and rollout are subsequent work. Organization scope and portable multi-agent packaging are confirmed requirements, not assumptions.

## 8. Related Specifications / Further Reading

Sources inspected during planning; recheck the target checkout before using their policies or scripts:

- [Fastify Ecosystem — Official Core Plugins](https://fastify.dev/ecosystem/): authoritative list of core plugins maintained by the Fastify team (60 official `@fastify/*` plugins; plugin repos not listed here are unmanned).
- [TSTyche Agent Skills Guide](https://tstyche.org/guides/agent-skills) and [`tstyche/tstyche-skills`](https://github.com/tstyche/tstyche-skills): official portable `SKILL.md` skills (`tstyche-type-tests`, `tstyche-project-setup`, `tstyche-programmatic-api`, `tstyche-migrate-from-tsd`) for writing, reviewing, and migrating TypeScript type tests.
- [Fastify contribution policy](https://github.com/fastify/fastify/blob/main/CONTRIBUTING.md): contribution conventions and human-opened PR requirement.
- [Fastify core package manifest](https://github.com/fastify/fastify/blob/main/package.json): evidence for repository-specific runtime/type-test commands.
- [fastify-plugin package manifest](https://github.com/fastify/fastify-plugin/blob/main/package.json): comparison demonstrating different command names and test composition.
- Current Fastify testing documentation retrieved through Context7 using the resolved `/fastify/fastify` library ID; target repository versions remain authoritative for implementation.
