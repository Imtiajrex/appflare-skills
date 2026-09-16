# Appflare Agent Skills

[Agent Skills](https://agentskills.io) that teach AI coding agents (Claude Code, Codex, Copilot, Cursor and others) how to build with **[Appflare](https://github.com/Imtiajrex/appflare)**, the generate-first backend framework for Cloudflare Workers.

## Install

```bash
npx skills add Imtiajrex/appflare-skills
```

To install a single skill:

```bash
npx skills add Imtiajrex/appflare-skills --skill appflare-schema
```

The [skills CLI](https://github.com/vercel-labs/skills) asks which agents to install into and copies the skills into their skill folders (for example `.claude/skills` or `.agents/skills`).

## Skills

| Skill | Use it when |
| --- | --- |
| [`appflare`](skills/appflare/SKILL.md) | Working in any Appflare project. Covers the mental model, the dev loop and which skill to load next |
| [`appflare-schema`](skills/appflare-schema/SKILL.md) | Adding or changing tables, columns, enums, JSON columns or relations, and running migrations |
| [`appflare-handlers`](skills/appflare-handlers/SKILL.md) | Writing `query`, `mutation`, `scheduler`, `cron` or `storageManager` handlers |
| [`appflare-querying`](skills/appflare-querying/SKILL.md) | Reading or writing data with `ctx.db`: filters, relations, pagination, atomic batches and transactions, SQL expressions, aggregates |
| [`appflare-client`](skills/appflare-client/SKILL.md) | Calling the backend from a frontend with the generated client, React hooks, realtime, auth or storage |
| [`appflare-cli-deploy`](skills/appflare-cli-deploy/SKILL.md) | Configuring `appflare.config.ts`, generating, migrating, adding admins and deploying |

Each skill is a short `SKILL.md` that loads when the skill activates, plus a `references/` folder the agent reads only when a task needs it.

## Without the CLI

The same skills ship inside the `appflare` npm package:

```bash
mkdir -p .claude/skills
cp -R node_modules/appflare/skills/appflare* .claude/skills/
```

## Documentation

Full docs: https://appflare-docs.imtiajbinaoual.workers.dev

These skills are maintained in the Appflare repository under [`packages/appflare/skills`](https://github.com/Imtiajrex/appflare/tree/main/packages/appflare/skills) and mirrored here.
