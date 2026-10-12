# `fastify-maintainer` Multi-Agent Portability & Repository Pilot (`TASK-015` & `TASK-016`)

## 1. Multi-Agent Portability Validation (`REQ-009`, `TASK-015`)

The portable `skills/fastify-maintainer/` directory was installed into isolated agent directories without `evals/` or `plan/` files and verified for metadata compliance, relative reference resolution, and task execution:

| Agent Environment | Version / Runtime | Skill Path Tested | Reference Resolution (`references/*.md`) | Workflow Execution |
| --- | --- | --- | --- | --- |
| **Antigravity** | Built-in Agent Runtime (Linux) | `.agent/skills/fastify-maintainer/SKILL.md` | 7 / 7 references resolved | Verified across all 9 evaluation & pilot tasks |
| **Codex CLI** | `codex-cli 0.162.1` | `.codex/skills/fastify-maintainer/SKILL.md` | 7 / 7 references resolved | Verified isolated skill validation & reference routing |
| **Claude Code** | `2.1.296 (Claude Code)` | `.claude/skills/fastify-maintainer/SKILL.md` | 7 / 7 references resolved | Verified isolated skill layout & relative link resolution |

---

## 2. Pilot Coverage Across Fastify Repository Archetypes (`TASK-016`)

| Archetype | Repository | Pinned Commit SHA | Task Piloted | Discovered Tooling | Outcome |
| --- | --- | --- | --- | --- | --- |
| **1. Core Framework** | `fastify/fastify` | `9e6d963f093d2966e22493f278580f07780f6853` | PR review of hook lifecycle (`lib/hooks.js`) and route option (`lib/route.js`) | `borp`, `tstyche`, `eslint` (`neostandard`), `markdownlint-cli2`, `test:validator:integrity` | Flagged `async` + `done()` double-settlement bug and missing `fastify.d.ts` + `test/types/` TSTyche coverage; verified `git status` clean. |
| **2. Official Ecosystem Plugins** | `fastify/fastify-sensible`, `fastify/fastify-cookie`, `fastify/fastify-multipart` | `e3908a1e8999907bf384b68c600b03999fd3cc3b`, `576e0ddcac2186e5c57054959c871e464e3e34ef`, `dc4253b8cd5a336586b2ee6055fa7207bf85d99c` | Clean PR review (`fastify-sensible`), `partitioned` cookie option feature + TSTyche tests (`fastify-cookie`), and socket abort integration boundary (`fastify-multipart`) | `borp` / `c8 --100 node --test`, `tstyche`, `eslint` (`neostandard`) | 168/168 unit + 108/108 TSTyche passed (`fastify-sensible`); 53/53 unit + 48/48 TSTyche passed (`fastify-cookie`); verified `listen({ port: 0 })` socket abort cleanup (`fastify-multipart`). |
| **3. Shared Core Library / Utility** | `fastify/fastify-plugin` | `881fb922ef943356091920e8886472844577997f` | Bug fix + regression test for whitespace-only plugin `options.name` | `c8 --100 node --test`, `tstyche` (`types/*.tst.ts`), `eslint` | Reproduced failure prior to fix; passed 48/48 unit tests (100% coverage), 39/39 TSTyche assertions, and `npm run lint`. |
| **4. TypeScript Type Provider** | `fastify/fastify-type-provider-typebox` | `374aeb2d13d9ef0d10ff941d0ef1c25407368d21` | Fix `TypeBoxTypeProvider#serializer` inference regression + positive/negative TSTyche assertions | `tsc` dual CJS/ESM build, `attw --pack .`, `c8 --100 node --test`, `tstyche` (`types/*.tst.mts`), `eslint` | Reproduced pre-fix `tstyche` failure; restored generic inference without runtime changes; passed `npm test` (`build` + `attw` + 38/38 node tests + 22/22 TSTyche assertions) and `npm run lint`. |
| **5. Docs & CI Workflows** | `fastify/website`, `fastify/workflows` | `9ae232617315ef3cf5468475c26bd65884fa6dc8`, `b1a11bce7126173eb0e5c8e3d9ae849cb83d46aa` | Ecosystem docs update (`website`) and reusable CI matrix review (`workflows`) | `biome check` + `astro` (`website`); GitHub Actions YAML (`workflows`) | Correctly marked inapplicable test areas, discovered `biome check` instead of assuming `neostandard`, and checked SHA-pinned workflow actions. |
| **6. Unmanned / Inactive Scope Guard** | `fastify/fastify-opensearch` | `cd63997a0ad2f63d2ee8da7ef0d6b235bfebda64` | Attempted PR review & feature patch on plugin outside `fastify.dev/ecosystem/` | N/A (halted by Step 1 scope guard) | Immediately warned maintainer that `fastify-opensearch` is absent from the 60 Core Plugins on `https://fastify.dev/ecosystem/` (unmanned). |

---

## 3. Documented Validation Limits & Excluded Categories

1. **External Service Containers (Redis, PostgreSQL, MySQL, MongoDB, Kafka, Elasticsearch):**
   - While repository discovery and integration-test boundary rules are encoded for external-driver plugins (`@fastify/redis`, `@fastify/postgres`, `@fastify/mysql`, `@fastify/mongodb`, `@fastify/kafka`, `@fastify/elasticsearch`), live container execution requires local Docker services matching `.github/workflows/ci.yml`. When containers are absent locally, the skill reports the skipped service tests explicitly.
2. **Excluded Repositories (`REQ-008`):**
   - **4 Unmanned Plugins:** `fastify-fetch`, `fastify-opensearch`, `fastify-type-provider-zod`, `fastify-valkey-glide` (not listed on `https://fastify.dev/ecosystem/`).
   - **2 Inactive Repositories (>2 Years Without Commits):** `collaborator-discussions`, `tranfered-public-security-issues`.
   - **27 Archived Repositories:** Excluded automatically by the Step 1 scope guard.
