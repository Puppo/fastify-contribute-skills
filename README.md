# Fastify Maintainer Agent Skills (`fastify-contribute-skills`)

Portable [Agent Skills](https://agentskills.io/specification) (`SKILL.md`) designed for [Fastify](https://github.com/fastify) maintainers to **review pull requests** and **implement features or bug fixes** across actively maintained repositories in the Fastify GitHub organization.

## Available Skills

- **[`fastify-maintainer`](skills/fastify-maintainer/SKILL.md)**: Repository-aware maintainer workflow with two operating modes (**Review** and **Implementation**) and conditional reference loading across all 8 maintainer areas:
  1. **Codebase Architecture** (`fastify-plugin` encapsulation, lifecycle, hooks, decorators, generated files)
  2. **JavaScript** (CJS/ESM conventions, `neostandard`, `@fastify/error` `FST_ERR_*` codes, `process-warning` `FSTDEP*`, hot-path performance, security boundaries)
  3. **TypeScript** (Synchronizing runtime `.js` with manual `.d.ts` declarations, `declare module 'fastify'` augmentation, generic propagation)
  4. **Runtime Tests** (`borp` / `node:test` + `@matteo.collina/tspl` `await p.completed`, `c8 --100` coverage, `tape`, legacy `tap`)
  5. **Type Tests** (**TSTyche** `.tst.ts` / `.test-d.ts`, `tsd`, and `tsc --project`)
  6. **Integration Tests** (Boundary selection: default in-process `fastify.inject()` vs ephemeral `fastify.listen({ port: 0 })` + `t.after(() => fastify.close())` vs external service fixtures)
  7. **CI** (`fastify/workflows` reusable workflows, Node/OS matrices, SHA-pinned actions, trusted publishing)
  8. **Documentation & Policy** (`README.md`, `docs/`, error/deprecation tables, `fastify/action-pr-title`, and `CONTRIBUTING.md` human-opened PR policies)

---

## Repository Scope Rules

The `fastify-maintainer` skill enforces two strict scope boundaries across `https://github.com/fastify`:

1. **Official Ecosystem Plugins Only:**
   - For Fastify plugins and type providers, **only** the 60 official Core Plugins listed on [https://fastify.dev/ecosystem/](https://fastify.dev/ecosystem/) are in scope.
   - Any plugin repository in the `fastify` organization not listed on `https://fastify.dev/ecosystem/` (such as `fastify-fetch`, `fastify-opensearch`, `fastify-type-provider-zod`, `fastify-valkey-glide`, or archived plugins) is considered **unmanned** and out of scope.
   - Standalone `@fastify/*` utilities that are not plugins (such as `@fastify/error`, `@fastify/busboy`, `@fastify/deepmerge`, `@fastify/send`, `@fastify/restartable`) are supported as shared libraries when active.
2. **2-Year Commit Activity Cutoff:**
   - Any repository (plugin or non-plugin) that is archived or has **no commits on its default branch within the last 2 years** (`git log -1 --since="2 years ago"`) is excluded as inactive.

See [evals/repositories.json](evals/repositories.json) and [skills/fastify-maintainer/references/repository-discovery.md](skills/fastify-maintainer/references/repository-discovery.md) for the complete audited inventory of all 143 `fastify` organization repositories (110 supported, 33 excluded).

---

## Installation

### Using `npx skills` (Multi-Agent Installer)

```bash
npx skills add <owner>/fastify-contribute-skills
```

### Recommended Companion Skills for Type Tests ([TSTyche](https://tstyche.org/guides/agent-skills))

Most active Fastify repositories use [TSTyche](https://tstyche.org/) for TypeScript type testing. Install the official TSTyche Agent Skills alongside `fastify-maintainer`:

```bash
npx skills add tstyche/tstyche-skills
```

When installed, `fastify-maintainer` delegates deep TSTyche authoring and `tsd`-to-TSTyche migration tasks to `tstyche-type-tests`, `tstyche-project-setup`, and `tstyche-migrate-from-tsd`, while retaining built-in TSTyche assertion rules in [`references/typescript.md`](skills/fastify-maintainer/references/typescript.md) for standalone installations.

### Manual / Isolated Installation by Coding Agent

Copy or symlink `skills/fastify-maintainer` into your coding agent's skill directory:

```bash
# Codex CLI (~/.codex/skills or project .agents/skills)
mkdir -p ~/.codex/skills
ln -s "$(pwd)/skills/fastify-maintainer" ~/.codex/skills/fastify-maintainer

# Claude Code (~/.claude/skills or project .claude/skills)
mkdir -p ~/.claude/skills
ln -s "$(pwd)/skills/fastify-maintainer" ~/.claude/skills/fastify-maintainer

# Antigravity (~/.gemini/antigravity-cli/skills or project .agent/skills)
mkdir -p .agent/skills
ln -s "$(pwd)/skills/fastify-maintainer" .agent/skills/fastify-maintainer
```

### Tested Agent Environments

| Coding Agent | Version Tested | Portable `SKILL.md` Discovery | Relative Reference Loading |
| --- | --- | --- | --- |
| **Antigravity** | Built-in Agent Runtime | Verified | Verified (`references/*.md`) |
| **Codex CLI** | `codex-cli 0.162.1` | Verified | Verified (`references/*.md`) |
| **Claude Code** | `2.1.296 (Claude Code)` | Verified | Verified (`references/*.md`) |

---

## Repository Layout

```text
.
├── README.md
├── plan/
│   └── feature-fastify-maintainer-skills-1.md   # Phased implementation plan & tracking
├── skills/
│   └── fastify-maintainer/
│       ├── SKILL.md                             # Portable skill entry point & router
│       └── references/
│           ├── repository-discovery.md          # Scope check (60 ecosystem plugins + 2y rule) & script discovery
│           ├── review.md                        # Read-only PR / diff review workflow
│           ├── implementation.md                # Reproduce-first feature & bug-fix workflow
│           ├── codebase-javascript.md           # Encapsulation, lifecycle, @fastify/error, performance, security
│           ├── typescript.md                    # JS + .d.ts sync, augmentation, TSTyche & tsd rules
│           ├── testing.md                       # borp / node:test + tspl, inject() vs listen({ port: 0 })
│           └── ci-docs.md                       # fastify/workflows, neostandard, docs & human-PR policy
└── evals/
    ├── repositories.json                        # Audited inventory of all 143 Fastify org repositories
    ├── pilot.md                                 # Pilot validation across Fastify repo archetypes
    └── fastify-maintainer/
        ├── cases.json                           # Curated review & implementation evaluation cases
        ├── triggers.json                        # Intended activation vs near-miss trigger cases
        └── report.md                            # With-skill vs baseline evaluation comparison report
```

---

## Usage Examples

### 1. Reviewing a Pull Request or Local Diff (Read-Only)
```text
Review the current branch diff against main in fastify/fastify. Check runtime hook semantics, .d.ts alignment, and whether integration tests are required.
```

### 2. Fixing a Bug with Regression Coverage
```text
Fix the bug in fastify-plugin where pre-release Fastify versions fail the version range check, add a regression test, and run the repository validation checks.
```

### 3. Implementing a Feature Across Runtime, `.d.ts`, Type Tests, and Docs
```text
Add support for the `partitioned` boolean option in @fastify/cookie, updating runtime serialization, types/index.d.ts, TSTyche type tests, unit tests, and README.md.
```

---

## Validation Limits & Contribution Policy

- **Local vs Matrix Validation:** The skill executes commands discovered from the checked-out repository (`package.json` scripts). When external services (Redis, PostgreSQL, MySQL, MongoDB, Kafka, Elasticsearch) or multi-OS/Node CI matrices are unavailable locally, the skill reports exact commands executed and explicitly lists unverified matrix/service checks rather than fabricating passes.
- **Human-Opened PR Policy:** In accordance with [Fastify's Contribution Policy](https://github.com/fastify/fastify/blob/main/CONTRIBUTING.md), maintainers and contributors are responsible for opening pull requests. The skill produces verified local patches, conventional PR titles (`fastify/action-pr-title`), and structured review/implementation summaries.