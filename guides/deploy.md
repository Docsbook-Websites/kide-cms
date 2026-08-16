---
title: Deploying Kide CMS to Node.js or Cloudflare
description: Deploy a Kide CMS project to Node.js with SQLite or Cloudflare Workers with D1 and R2, including scheduled publishing and environment variables.
---

# Deploy

Kide builds on Astro and deploys anywhere Astro runs. The scaffolder wires one of two targets — **Node.js** (SQLite + local files) or **Cloudflare Workers** (D1 + R2). Both distribution modes (embedded or package, see [Getting Started](../getting-started.md)) deploy identically.

|  | Node.js | Cloudflare |
| --- | --- | --- |
| Database | SQLite at `data/cms.db` | D1 (`CMS_DB` binding) |
| Assets | `public/uploads/` | R2 (`CMS_ASSETS` binding) |
| Images | Sharp, on-demand + cached | Cloudflare Image Transformations |

Two one-line adapter files handle the difference — `src/cms/adapters/db.ts` and `storage.ts` — and the scaffolder already sets them.

## Cloudflare

`create-kide-app` offers to provision everything during scaffolding: D1 database, R2 bucket, migrations, and the first deploy. If you accepted, you're already live. Skipped provisioning? Four commands:

```bash
pnpm dlx wrangler d1 create my-site-db            # paste the id into wrangler.toml
pnpm dlx wrangler r2 bucket create my-site-assets
pnpm dlx wrangler d1 migrations apply my-site-db --remote
pnpm run deploy
```

The generated `wrangler.toml` ships a placeholder `database_id` so `pnpm dev` works immediately (local D1/R2 are emulated) — only deploying needs the real id.

**Schema changes.** After editing collections, run `pnpm db:generate` and commit the migration file it writes to `src/cms/migrations/`. In CI, apply migrations before the build:

```bash
pnpm dlx wrangler d1 migrations apply my-site-db --remote && pnpm build
```

The `--remote` flag matters — without it, wrangler migrates your *local* D1 and production stays on the old schema.

**Scheduled publishing** works without setup. A cron trigger fires every minute and the worker handles publishing and background tasks. Secure the endpoints:

```bash
pnpm dlx wrangler secret put CRON_SECRET
```

**Images.** Enable **Images → Transformations → Resize images from any origin** in the Cloudflare dashboard. Without it, images serve at full size — everything else still works.

## Node.js

```bash
pnpm build
pnpm cms:push                  # sync schema — fails loudly, stopping a broken deploy
node dist/server/entry.mjs
```

The server does no schema work at boot, so `cms:push` belongs in every deploy. Additive changes apply directly. An ambiguous rename/drop is refused with guidance: run `pnpm cms:push --recreate=<collection> --allow-data-loss` to drop and recreate the named tables (plus their `_translations`/`_versions` tables), or hand-write a migration to preserve data.

**Persistent storage.** If deploys replace the app directory (containers, fresh releases), point these at a location outside it and back that place up:

```bash
CMS_DATABASE_URL=/srv/kide/shared/cms.db     # default: ./data/cms.db
CMS_UPLOADS_DIR=/srv/kide/shared/uploads     # default: ./public/uploads
```

Have your web server (Caddy, nginx) serve `/uploads/*` straight from `CMS_UPLOADS_DIR`.

**Compression.** The default `node({ mode: "standalone" })` adapter serves assets uncompressed — fine behind a CDN or reverse proxy that compresses. On a bare VM with nothing in front, switch to `node({ mode: "middleware" })` in `astro.config.mjs` and wrap the handler in a small Express server:

```js
// server.mjs
import express from "express";
import compression from "compression";
import { handler } from "./dist/server/entry.mjs";

const app = express();
app.use(compression());
app.use("/_astro", express.static("dist/client/_astro", { immutable: true, maxAge: "1y" }));
app.use(express.static("dist/client"));
app.use(handler);
app.listen(4321);
```

**Scheduled publishing.** Nothing polls the cron endpoints on Node — that's the Cloudflare Worker's `scheduled()` handler. Self-hosted deployments must poll *both* endpoints once a minute or scheduled publishing and background tasks never run:

```bash
* * * * * curl -s -H "Authorization: Bearer $CRON_SECRET" https://example.com/api/cms/cron/publish
* * * * * curl -s -H "Authorization: Bearer $CRON_SECRET" https://example.com/api/cms/cron/tasks
```

## Environment variables

| Variable | Required | Description |
| --- | --- | --- |
| `CRON_SECRET` | Required | Secures the cron endpoints (`401` in production until set) |
| `CMS_TRUSTED_ORIGIN` | Recommended | Canonical public origin used by admin CSRF checks, e.g. `https://cms.example.com` |
| `RESEND_API_KEY` | Optional | Enables automatic invite emails via Resend |
| `RESEND_FROM_EMAIL` | Optional | Email sender address |
| `AI_PROVIDER` | Optional | AI provider for content generation |
| `AI_API_KEY` | Optional | AI provider API key |
| `AI_MODEL` | Optional | AI model name |
| `WEBHOOK_SECRET_<PROVIDER>` | Optional | HMAC secret for inbound webhooks at `/api/cms/webhooks/<provider>` (hyphens in the provider name map to underscores) |

**Node.js**: put them in `.env`. **Cloudflare**: secrets via `pnpm dlx wrangler secret put <NAME>`, non-secret values under `[vars]` in `wrangler.toml`.

> **Reading secrets on Cloudflare** — `import.meta.env` cannot see runtime secrets on Workers, so checking it makes AI and email silently appear disabled. Inside CMS code, always use the `readEnv()` helper — it reads from the Worker environment correctly on every target.
