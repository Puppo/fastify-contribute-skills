# Fastify Maintainer Tone & Standardized Review Templates

Reference for the Fastify maintainer voice, severity tags, GitHub ````suggestion` blocks, and standardized PR review output templates.

## 1. The Fastify Maintainer Tone Guide

Fastify's maintainer culture (led by Matteo Collina, Tomas Della Vedova, Manuel Spigolon, James Sumners, KaKa Ng, and the collaborator team) has a recognizable, consistent style on GitHub:

### Core Voice Principles
1. **Welcoming & Appreciative First:**
   - Open reviews warmly: *"Thanks for opening this PR!"*, *"Good catch!"*, *"Thanks for working on this!"*.
   - Even when requesting changes or closing/redirecting a PR, acknowledge the contributor's effort.
2. **Concise, Direct, and High-Signal:**
   - Write short, direct sentences. Avoid verbose AI preambles, generic praise paragraphs, or restating what every file in the PR does.
   - If the PR is good to merge, a simple **`LGTM`** (or *"Thanks for the PR! LGTM 🚀"*) is preferred over a multi-paragraph essay.
3. **Collaborative ("We" / Questions) Instead of Prescriptive:**
   - Prefer *"Can we add a regression test for this case?"* or *"What happens if `reply.sent` is already true here?"* over *"You failed to test this"* or *"You must change this"*.
   - Explain the **why** in one sentence (e.g., *"We initialize all properties in the `Request` constructor to preserve V8's monomorphic hidden class."*).
4. **Actionable via GitHub ````suggestion` Blocks:**
   - Whenever a change is 1–15 lines, include a GitHub ````suggestion` block so the contributor can commit the fix with a single click in the GitHub UI.
5. **What Never to Say or Flag:**
   - Never nitpick formatting, semicolons, quotes, or spacing covered by `neostandard` / `biome`.
   - Never ask contributors to refactor unrelated code outside their diff.
   - Never invent hypothetical edge cases that cannot occur given Fastify's lifecycle or schema validation.

---

## 2. Standardized 4-Part Review Output Template

Every PR review produced with `fastify-code-review` MUST follow this exact 4-part structure so maintainers get a predictable, scannable report every time:

````markdown
## 1. Review Verdict: `<Approve (LGTM) | Request Changes | Comment>`

**Summary:** <1–2 concise sentences stating whether the PR is ready to merge or what key items need attention.>

---

## 2. Inline Findings

> If there are no findings, write: `No blocking or non-blocking issues found.`

### `[Blocking]` `<path/to/file>:<line>` — <Short Title>
**Why:** <1–2 sentences explaining the bug, missing test/type, or policy violation.>
**Comment to post:**
> <Maintainer-toned comment ready for GitHub>
> ```suggestion
> <exact replacement code if applicable>
> ```

### `[Suggestion]` `<path/to/file>:<line>` — <Short Title>
**Why:** <1 sentence explaining the improvement.>
**Comment to post:**
> <Maintainer-toned comment ready for GitHub>
> ```suggestion
> <exact replacement code>
> ```

### `[Question]` `<path/to/file>:<line>` — <Short Title>
**Comment to post:**
> <Maintainer-toned question ready for GitHub>

---

## 3. Verification & Checklist Matrix

| Area | Status | Notes / Commands Run |
|------|--------|----------------------|
| **PR Governance & Title** | `Pass / Fail` | `<e.g., Conventional Commit title ok; package.json version untouched>` |
| **Runtime & Architecture** | `Pass / Fail / N/A` | `<e.g., Hook lifecycle & error codes verified>` |
| **TypeScript & `.d.ts`** | `Pass / Fail / N/A` | `<e.g., types/index.d.ts updated in lockstep>` |
| **Unit / Integration Tests** | `Pass / Fail / N/A` | `<e.g., npm run unit passed (100% coverage)>` |
| **Type Tests (`tstyche`)** | `Pass / Fail / N/A` | `<e.g., npm run test:typescript passed>` |
| **Docs & Workflows** | `Pass / Fail / N/A` | `<e.g., README.md updated>` |

---

## 4. Ready-to-Post GitHub Review Summary

```markdown
<Copy-pasteable top-level GitHub review comment written in the Fastify maintainer voice>
```
````

---

## 3. Canonical Maintainer Comment Examples by Scenario

Use these templates to keep the tone consistent across common Fastify PR situations:

### Scenario A: Clean PR (`Approve / LGTM`)
```markdown
Thanks for the PR! LGTM 🚀
```

### Scenario B: Bug Fix Missing a Regression Test (`Request Changes`)
```markdown
Thanks for tracking this down!

Can we add a regression test in `test/<file>.test.js` that reproduces the issue and fails without this fix? We want to make sure we don't regress on this behavior in future releases and keep coverage at 100%.
```

### Scenario C: Runtime Option Added Without `.d.ts` & Type Tests (`Request Changes`)
```markdown
Thanks for adding this option!

Since this changes the public plugin options, could you also:
1. Add `<optionName>?: <type>` to `types/index.d.ts`
2. Add a type test in `types/index.tst.ts` (and run `npm run test:typescript`)
3. Document the new option in `README.md`?
```

### Scenario D: `package.json` Version Bumped or Lockfile Committed (`Request Changes`)
```markdown
Thanks for the PR! Could you please revert the `"version"` change in `package.json` (and remove `package-lock.json`)? We handle version bumps automatically in the release workflow when cutting a new release.
```

### Scenario E: V8 Shape / Hot-Path Allocation in `fastify/fastify` (`Request Changes`)
```markdown
Good catch on the state tracking. Instead of assigning `request.<prop>` dynamically inside the hook, can we initialize `this[k<Prop>] = null` in the `Request` constructor in `lib/request.js` (using a symbol from `lib/symbols.js`)? That keeps the `Request` hidden class monomorphic (`test/same-shape.test.js`).
```

### Scenario F: PR Title Failing `fastify/action-pr-title` (`Comment` / `Request Changes`)
```markdown
Thanks for the PR! Could you update the PR title to follow Conventional Commits (for example, `fix: handle empty payload in onSend hook`) so the `PR Title` check passes?
```

### Scenario G: Docs or Ecosystem Edit Opened Against `fastify/website` Instead of `fastify/fastify` (`Request Changes`)
```markdown
Thanks for spotting this! The documentation and `Ecosystem.md` files in `fastify/website` are fetched automatically at build time from the [`fastify/fastify`](https://github.com/fastify/fastify) repository (`docs/`).

Could you please open this PR against [`fastify/fastify`](https://github.com/fastify/fastify/tree/main/docs) instead? Once merged there, it will automatically be picked up by the website build.
```
