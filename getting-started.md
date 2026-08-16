---
title: Getting Started with Kide CMS
description: Scaffold a new Kide CMS project with pnpx create-kide-app, choose a runtime mode, and pick a deploy target for Node.js or Cloudflare Workers.
---

# Getting Started

Kide CMS scaffolds a full Astro project with the CMS wired in. Supports Astro's route caching with tag-based invalidation, so cached pages stay correct when content changes.

```bash
pnpx create-kide-app
```

The scaffolder then asks two questions: how the runtime should live in your project, and where you plan to deploy.

<!-- widget:stepper -->

### Choose a runtime mode

**Package** — recommended for most projects. The runtime is an `@kidecms/core` npm dependency, so most updates are a version bump. If you outgrow it, `pnpm exec kide eject` converts the project to embedded in place.

**Embedded** — gives you full control. The CMS source scaffolds into `src/cms/` as part of your project, so you can read, modify, and audit it directly, with no plugin API in the way. Upgrades arrive as patches you review and apply.

Both modes share the same runtime, API, and admin UI — the difference is only where the runtime lives and how updates arrive.

### Choose a deploy target

**Node.js** — runs anywhere Node runs, using SQLite for storage.

**Cloudflare** — deploys as a Worker; provisions D1 (database) and R2 (assets) for you.

See [Deploy](./guides/deploy.md) for the full comparison and day-2 operations for each target.

<!-- /widget -->

## Commands

| Command | What it does |
| --- | --- |
| `pnpm dev` | Start dev server (auto-generates schema, pushes to DB) |
| `pnpm build` | Production build |
| `pnpm check` | astro check + Cloudflare TS profile + eslint |
| `pnpm test` | Run the test suite (vitest) |
| `pnpm format` | Format with Prettier |
| `pnpm cms:generate` | Regenerate `.generated/` from `cms.config.ts` |
| `pnpm cms:admin` | Create an admin user from the CLI |
| `pnpm cms:mcp` | Start the local MCP server for AI-assisted editing |
| `pnpm cms:push` | Sync the local SQLite DB to the generated schema |
| `pnpm cms:upgrade` | Upgrade to a newer Kide release |
| `pnpm cms:restore` | Roll back a failed upgrade from its backup |
| `pnpm db:generate` | Generate migration SQL from schema changes (D1) |

In dev mode, schema changes push to the database automatically. On the Node target, `cms:push` is the same sync used at deploy time; migration files are only needed for Cloudflare D1.

## Project structure

```
src/cms/
  cms.config.ts           # CMS config (locales, admin, collection imports)
  collections/             # One file per collection (schema, access rules)
  adapters/                # db / email / storage adapters (project-owned)
  fields/                  # Custom admin field components (optional)
  runtime.ts               # Wires adapters into the core runtime
  .generated/               # Auto-generated — don't edit
    schema.ts               # Drizzle tables
    types.ts                # TypeScript interfaces
    validators.ts           # Zod schemas
    api.ts                  # Typed API
  core/, admin/, routes/    # CMS runtime (embedded mode only —
  middleware/, platform/    # in package mode these ship inside @kidecms/core)
```

## Environment variables for local development

Set these in `.env`. The full table with production requirements lives on the [Deploy](./guides/deploy.md) page.

| Variable | Description |
| --- | --- |
| `AI_PROVIDER` | AI provider (`openai`) |
| `AI_API_KEY` | AI provider API key |
| `AI_MODEL` | AI model name (e.g. `gpt-4o-mini`) |
| `RESEND_API_KEY` | Enables automatic invite emails for new users |
| `RESEND_FROM_EMAIL` | Email sender address |
| `CMS_TRUSTED_ORIGIN` | Canonical public origin for admin CSRF checks |
| `CRON_SECRET` | Secures the cron endpoints (publish, tasks) |

## Next step

Once the project is running, define your first collection — see [Collections](./guides/collections.md).
