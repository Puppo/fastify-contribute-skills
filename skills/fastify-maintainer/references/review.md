# Pull Request & Diff Review Workflow

Use this workflow when asked to review a pull request, branch, commit, or local diff in a Fastify organization repository.

> **Hard Constraint (`CON-003`):** Review mode is strictly non-mutating. Do **not** leave edits in tracked files or create commits.

## 1. Establish the Diff & Target Contract

1. **Complete Repository Discovery:** Follow [repository-discovery.md](repository-discovery.md) first. If the repository is an unmanned plugin (not listed on `https://fastify.dev/ecosystem/`) or has had no commits in the last 2 years, flag that immediately at the top of the review.
2. **Determine Base and Head:**
   - If reviewing a PR or branch, identify the base branch (`main`, `master`, or maintenance branch like `v4.x`) and run `git merge-base <base> <head>` followed by `git diff <merge-base>...<head>`.
   - If reviewing uncommitted local changes, inspect `git status --short` and `git diff` / `git diff --cached`.
   - If the comparison target cannot be determined from the workspace or user prompt, stop and state the missing input rather than guessing.
3. **Determine Area Applicability:**
   - Map changed files and public contracts against the 8 maintainer areas (`Codebase`, `JavaScript`, `TypeScript`, `Runtime tests`, `Type tests`, `Integration tests`, `CI`, `Docs`).
   - Load **only** the conditional reference files relevant to the diff:
     - Runtime JS or architecture changes → [codebase-javascript.md](codebase-javascript.md)
     - `.d.ts`, `.ts`, public API signature changes, or type tests → [typescript.md](typescript.md)
     - Test additions/modifications or missing coverage → [testing.md](testing.md)
     - Workflow YAML, `package.json` scripts/engines, or documentation → [ci-docs.md](ci-docs.md)

## 2. Inspect for High-Signal Maintainer Issues

Focus on concrete behavioral, compatibility, type-safety, security, and performance defects—not cosmetic preferences already handled by the repository's linter:

- **Contract & Compatibility Drift:**
  - Does a runtime JS change add/modify an option, decorator, hook, or return value without updating `.d.ts` declarations (`index.d.ts`, `types/index.d.ts`, `fastify.d.ts`), type tests, and `README.md` / `docs/`?
  - Does the change break backward compatibility on the target branch (e.g., changing error codes, removing overload signatures, altering default options)?
  - In `fastify/fastify`, if `lib/config-validator.js` or `lib/error-serializer.js` sources changed, was `npm run test:validator:integrity` respected?
- **Fastify Runtime & Lifecycle Hazards:**
  - Are `async` functions and `done` callbacks mixed in hooks, plugins, or custom parsers (risking double invocation or hanging requests)?
  - Is `fastify-plugin` (`fp`) used appropriately (or omitted when encapsulation is intended)?
  - Are errors created via `@fastify/error` (`FST_ERR_*`) rather than untyped `new Error()` where the repository standardizes error codes?
  - Does request-time code allocate closures, compile schemas/regexes, or mutate shared state that belongs at registration time?
- **Test & Type-Test Sufficiency:**
  - Does a bug fix include a regression test that would fail without the fix?
  - Do tests clean up resources (`t.after(() => fastify.close())`) and await completion (`await t.completed` when using `@matteo.collina/tspl` with callbacks)?
  - Are integration boundaries appropriate (`fastify.inject()` by default; real `listen({ port: 0 })` or external service fixtures only when transport/socket/stream/driver boundaries matter)?
  - Do type tests (`tstyche` or `tsd`) check both positive inference and negative/invalid usage where appropriate?

## 3. Validate Locally (Without Mutating Tracked Files)

Where dependencies and environment allow:
1. Run the repository's discovered linter, targeted runtime test file(s), and type-test command (`npm run test:types` / `npm run test:typescript` / `npx tstyche <file>`).
2. Confirm `git status --short` is unchanged after running checks.

## 4. Required Review Output Structure

Format every review response using these sections:

1. **Review Context & Scope:**
   - Repository name, scope status (`Supported` vs `Unmanned plugin` / `Inactive >2y`), and reviewed `base...head` (or local diff).
2. **Coverage & Applicability Matrix:**
   - Concise table or list covering all 8 areas (`Codebase`, `JavaScript`, `TypeScript`, `Runtime tests`, `Type tests`, `Integration tests`, `CI`, `Docs`), marking each as `Checked` or `Not Applicable (reason)`, plus the explicit **Integration Test Rationale**.
3. **Actionable Findings (Defects & Blockers):**
   - For each defect:
     - **Severity:** `Blocker` | `Major` | `Minor`
     - **Location:** `path/to/file:line`
     - **Trigger & Impact:** How the bug, type mismatch, or regression is triggered and what breaks.
     - **Recommended Fix:** Concrete code or test change required.
   - *Clean PR Rule:* If no defects exist, explicitly write: **`No actionable defects found.`** Do not invent artificial blockers.
4. **Optional Suggestions (Non-Blocking):**
   - Clearly separated minor improvements or readability notes.
5. **Validation & Limitations:**
   - Exact commands executed and their exit statuses, plus any checks that could not run (e.g., unavailable Redis/Postgres service or remote CI matrix).
