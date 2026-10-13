---
name: fastify-workflows
description: Use when maintaining, reviewing pull requests, adding or updating reusable workflows, matrices, service containers, security policies, or documentation in the fastify/workflows repository (@fastify/workflows). Covers all reusable CI workflows (plugins-ci.yml, nested-quality.yml, plugins-ci-redis/postgres/mysql/mongo/kafka/elasticsearch/package-manager.yml), reusable-release.yml, reusable-ossf-scorecard.yml, lock-threads.yml, internal validation (action-validator in validate-workflows-files.yml), SHA pinning, shell-injection hardening, and optic tag-sync releases. Do not use for individual Fastify plugin repositories (use fastify-plugins instead).
---

# Fastify Workflows Repository Maintainer (`fastify/workflows`)

Use this skill when maintaining, reviewing pull requests, or implementing changes inside the **[`fastify/workflows`](https://github.com/fastify/workflows)** repository (`@fastify/workflows`).

> **Boundary Note:** This skill is specifically for maintaining the centralized `fastify/workflows` repository. When reviewing a pull request in `fastify/workflows`, pair this skill with [`fastify-code-review`](../fastify-code-review/SKILL.md). When configuring or maintaining an individual `@fastify/*` plugin repository that *calls* these workflows, use [`fastify-plugins`](../fastify-plugins/SKILL.md) instead.

---

## Step 1: Recognize `fastify/workflows` Repository Characteristics

Before reviewing a diff or editing files in `fastify/workflows`, observe these repository truths:

1. **Workflow-Only Repository (No Runtime JS/TS or `npm test`):**
   - `package.json` (`"name": "@fastify/workflows"`, `"private": true`) contains `"scripts": {}`, `"dependencies": {}`, and `"devDependencies": {}`.
   - There are **no** JavaScript source files, TypeScript declarations, unit tests, type tests, or integration tests in this repository. Never invent `npm test` or `npm run lint` commands.
2. **Every Reusable Workflow Change Has Organization-Wide Blast Radius:**
   - Official Fastify plugins and libraries call `.github/workflows/plugins-ci*.yml`, `reusable-release.yml`, and `lock-threads.yml`.
   - Removing or renaming a `workflow_call` input, changing a default value incompatibly, or tightening required caller permissions without a major version bump breaks CI across the entire Fastify organization.

---

## Repository Layout (`fastify/workflows`)

```text
.
├── .editorconfig
├── .gitattributes
├── .gitignore
├── CONTRIBUTING.md                                      # feat/ branches, American English, sentence-case headings, optic release
├── LICENSE                                              # MIT
├── package.json                                         # @fastify/workflows (private: true, scripts: {})
├── README.md                                            # Usage examples, input tables, trusted publishing guide
└── .github/
    ├── dependabot.yml                                   # Monthly github-actions updates (prefix: ci, 7-day cooldown)
    └── workflows/
        ├── lock-threads.yml                             # Reusable: locks inactive issues/PRs after 90 days
        ├── nested-quality.yml                           # Reusable helper: dependency-review, license-check, linter
        ├── ossf-scorecard.yml                           # Internal: runs reusable-ossf-scorecard.yml on fastify/workflows
        ├── plugins-ci.yml                               # Reusable: core multi-OS (ubuntu/macos/windows) & Node matrix CI
        ├── plugins-ci-elasticsearch.yml                 # Reusable: Elasticsearch container CI
        ├── plugins-ci-kafka.yml                         # Reusable: Zookeeper + Kafka container CI
        ├── plugins-ci-mongo.yml                         # Reusable: MongoDB container CI
        ├── plugins-ci-mysql.yml                         # Reusable: MySQL container CI
        ├── plugins-ci-package-manager.yml               # Reusable: pnpm & yarn cross-package-manager CI
        ├── plugins-ci-postgres.yml                      # Reusable: PostgreSQL container CI
        ├── plugins-ci-redis.yml                         # Reusable: Redis 5/6/7/8 container matrix CI
        ├── release.yml                                  # Internal: releases fastify/workflows & syncs vX/vX.Y tags via optic
        ├── reusable-ossf-scorecard.yml                  # Reusable: OSSF Scorecard SARIF analysis & upload
        ├── reusable-release.yml                         # Reusable: npm OIDC provenance publish + GitHub release for plugins
        └── validate-workflows-files.yml                 # Internal CI: validates all .github/workflows/*.yml via action-validator
```

---

## Step 2: Load the Relevant Reference

Load the reference file(s) matching your task in `fastify/workflows`:

| Workflow Area | When to Load | Reference |
| --- | --- | --- |
| **1. Reusable CI & Service Workflows** | Modifying `plugins-ci.yml`, `nested-quality.yml`, `plugins-ci-package-manager.yml`, or any `plugins-ci-<service>.yml` (`redis`, `postgres`, `mysql`, `mongo`, `kafka`, `elasticsearch`)—including Node.js default versions, OS matrices, service container versions/healthchecks, `license-check`, or Dependabot `automerge`. | [references/ci-workflows.md](references/ci-workflows.md) |
| **2. Reusable Release, Security & Internal Repo Workflows** | Modifying `reusable-release.yml` (npm OIDC trusted publishing, `release` environment gate, `release:build` hook), `reusable-ossf-scorecard.yml`, `lock-threads.yml`, or `fastify/workflows`'s own internal workflows (`release.yml`, `ossf-scorecard.yml`, `validate-workflows-files.yml`, `.github/dependabot.yml`). | [references/release-and-maintenance.md](references/release-and-maintenance.md) |
| **3. Authoring, Security, Validation & Docs** | Reviewing or validating workflow YAML syntax (`action-validator`), checking 40-char SHA pinning, shell-injection safety in `run:` blocks, `permissions:` blocks, `persist-credentials`, `README.md` input tables, or `CONTRIBUTING.md` conventions. | [references/validation-and-security.md](references/validation-and-security.md) |

---

## Core `fastify/workflows` Maintainer Rules

1. **Synchronize Shared Inputs & Node Defaults Across All `plugins-ci*.yml` Files:** When updating default `node-versions` (e.g., `'["24", "26"]'`), `actions/checkout`, `actions/setup-node`, or `fastify/github-action-merge-dependabot`, update `plugins-ci.yml`, `plugins-ci-package-manager.yml`, and all 6 `plugins-ci-<service>.yml` files consistently.
2. **40-Character Commit SHA Pinning:** Every external `uses:` action must be pinned to a full 40-character commit SHA with a trailing version comment (`# vX.Y.Z`). Internal calls within `fastify/workflows` use `./.github/workflows/<file>.yml`.
3. **Validate with `action-validator` & Sync `README.md`:** Every workflow change must pass `action-validator` (as enforced by `validate-workflows-files.yml`) and update the corresponding input/usage tables in `README.md`.
