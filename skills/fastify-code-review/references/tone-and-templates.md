# Fastify Maintainer Tone & Review Templates

Reference for concise maintainer comments, severity tags, verified GitHub suggestion blocks, and review output.

## 1. Fastify Maintainer Tone

- Be welcoming, direct, and collaborative. A brief thanks at the start is enough; do not repeat praise in every finding.
- Write short sentences. State the concrete impact and requested change once; skip AI preambles and file-by-file summaries.
- Prefer “Can we add a regression test for this case?” over “You failed to test this.” Explain why in one sentence.
- Use GitHub ````suggestion` blocks only for verified, self-contained replacements mapped to exact changed line ranges. Otherwise describe the required behavior or test.
- Avoid formatting nitpicks covered by the repository's linter, unrelated refactors, and hypothetical cases that cannot occur under the actual lifecycle or schema.

## 2. Compact Review Output

Default to a verdict, findings only when present, and brief verification evidence. Write findings directly as ready-to-post comments, ordered `[Blocking]`, `[Suggestion]`, then `[Question]`. Do not repeat them in a separate summary or add empty sections. Expand into a detailed report or checklist matrix only when requested.

```markdown
<Request Changes | Comment | Approve (LGTM)> — <brief reason if needed>

[Blocking] <file:line> — <concrete impact>. <requested change>

Verification: <repository / PR URL or range>; base <SHA>, head <SHA>.
Local: <tested SHA; clean/isolated checkout or working-tree patch scope>; <exact commands and exit codes>.
Coverage: <command, exit code, metrics and thresholds, or Not run with reason>.
Remote CI: <run link, checked SHA, head/merge revision, status, or Not run with reason>.
```

Include only relevant verification categories and combine short entries when useful. Keep applicable but unavailable checks visible as `Not run` with a reason; omit categories outside the change's scope. Distinguish local tests, type tests, coverage gates, and remote CI. Unit-test success alone is not coverage evidence.

Put material verification limitations in the verdict, including an unavailable matching checkout. Environment failures are not demonstrated PR failures; use `Comment` when limitations prevent a supported approval. Preserve the read-only review and posting-authorization rules in the companion workflow.

### Clean Review Example

Illustrative values below must be replaced with actual evidence:

```markdown
Approve (LGTM). Thanks for the PR!

Verification: fastify/example-plugin PR #42; base <SHA>, head <SHA>.
Local at <head SHA>, isolated checkout: `npm run test:unit`, `npm run test:typescript`, `npm run lint` — exit 0 each.
Coverage: <configured command> — exit 0; <actual metrics / thresholds>.
Remote CI: <run link> at <head SHA> — passed.
```

### One Finding with a Verification Limitation

```markdown
Request Changes — the fix needs regression coverage.

[Blocking] test/example.test.js:24 — This test also passes without the fix. Can we add the failing-input case to protect against regression?

Verification: fastify/example-plugin PR #42; base <SHA>, head <SHA>.
Local at <head SHA>, isolated checkout: `npm run test:unit` — exit 0.
Coverage: Not run — required service unavailable.
Remote CI: <run link> at <head SHA> — pending.
```

## 3. Maintainer Comment Examples

Use these as short comment patterns, with actual diff locations and repository evidence. The review's opening thanks need not be repeated.

| Scenario | Comment |
| --- | --- |
| Bug fix missing a regression test | Can we add a test that fails without this fix? |
| Public option missing types and docs | Can we add this option to the public declarations, cover it in a type test, and document it? |
| Ordinary PR violates confirmed version policy | Could you revert the version bump? This repository reserves it for the release process. |
| Newly added lockfile violates confirmed policy | Could you remove the new lockfile? This repository's contribution rules exclude it. |
| Request property added dynamically | Can we initialize this property in the `Request` constructor using the repository's symbol convention? That preserves its monomorphic shape. |
| PR title fails the configured check | Could you update the title to match this repository's title check? |
| Website PR edits fetched docs | Could you open this change against `fastify/fastify/docs/`? The website fetches these files from there. |
| Proposed core feature fits a plugin | Can we implement this through the existing hook in a plugin? That keeps core minimal. |
| RFC interpretation is unresolved | Which revision of the linked RFC does this target? The newer proposal changes this requirement. |

Apply policy comments only after confirming local rules. Version comments do not apply to documented release PRs; lockfile comments do not apply where the file is tracked by policy.
