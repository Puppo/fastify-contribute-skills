# Fastify Maintainer Agent Skills

Portable [Agent Skills](https://agentskills.io/specification) for maintaining Fastify repositories and reviewing pull requests with a consistent flow and concise, collaborative tone.

## Choose a skill

| Skill | Use it for |
|-------|------------|
| [fastify-code-review](skills/fastify-code-review/SKILL.md) | Reviewing PRs and diffs; pairs with the relevant repository skill below. |
| [fastify-core](skills/fastify-core/SKILL.md) | Framework code, tests, public types, documentation, and workflows in `fastify/fastify`. |
| [fastify-plugins](skills/fastify-plugins/SKILL.md) | Ecosystem plugins, type providers, and standalone utilities: structure, implementation, tests, types, CI, and releases. |
| [fastify-workflows](skills/fastify-workflows/SKILL.md) | Reusable CI and release workflow definitions in `fastify/workflows`. |
| [fastify-website](skills/fastify-website/SKILL.md) | Website components, styling, data generation, and deployment in `fastify/website`. |

Each skill has a `SKILL.md` entrypoint and focused guides in its `references/` directory. Repository configuration and contribution rules determine which conventions apply. Plugin-specific rules do not apply automatically to standalone utilities.

## Installation

Install with the multi-agent skills installer:

```bash
npx skills add Puppo/fastify-contribute-skills
```

For manual installation, copy the desired folders from `skills/` into your agent's skill directory. To link all five from a local clone, run this from the repository root and replace the destination with your agent's directory:

```bash
skill_destination=/absolute/path/to/your/agent/skills
mkdir -p "$skill_destination"
for skill in skills/*; do
  ln -s "$(pwd)/$skill" "$skill_destination/$(basename "$skill")"
done
```

Install `fastify-code-review` alongside the repository skills you want to review with.

### Companion skills

For advanced type-test authoring in `fastify-core` and `fastify-plugins`, install the [TSTyche skills](https://tstyche.org/guides/agent-skills):

```bash
npx skills add tstyche/tstyche-skills
```

For `fastify-website`, the [`fastify/website`](https://github.com/fastify/website) repository already bundles [`frontend-design`](https://github.com/fastify/website/blob/main/.agents/skills/frontend-design/SKILL.md) and [`web-design-guidelines`](https://github.com/fastify/website/blob/main/.agents/skills/web-design-guidelines/SKILL.md) under [`.agents/skills/`](https://github.com/fastify/website/tree/main/.agents/skills) (tracked in [`skills-lock.json`](https://github.com/fastify/website/blob/main/skills-lock.json)). If you are reviewing `fastify/website` pull requests from outside a local checkout, you can also install them globally:

```bash
npx skills add anthropics/skills --skill frontend-design
npx skills add vercel-labs/agent-skills --skill web-design-guidelines
```

## Examples

- **Review:** “Review this PR in fastify/fastify using fastify-code-review and fastify-core. Include findings, verification evidence, and a ready-to-post summary.”
- **Plugin:** “Use fastify-plugins to add an option, including relevant runtime tests, public declarations, type tests, and documentation.”
- **Utility:** “Use fastify-plugins to fix this utility's behavior while preserving its exported API and existing tooling.”
- **Workflows:** “Use fastify-workflows to update the Node test matrix, check caller compatibility, and synchronize input documentation.”
- **Website:** “Use fastify-website to update a sponsor entry and its page layout, then validate the build and accessibility.”

## Review and validation behavior

Reviews preserve the user's working tree. They record the repository and reviewed revisions, verify that local checks test the correct commit, and use isolated temporary checkouts when necessary. Reports distinguish local tests, coverage gates, and remote CI; unavailable checks are marked `Not run` with a reason.

Implementation follows the target repository's contribution and release process. Review comments are prepared locally; posting them requires an explicit request. Follow the target repository's rules about human-opened PRs, including [Fastify core's contribution policy](https://github.com/fastify/fastify/blob/main/CONTRIBUTING.md).
