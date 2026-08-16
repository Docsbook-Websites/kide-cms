---
title: Kide CMS — Code-First CMS for Astro
description: Generated documentation preview for Kide CMS, a code-first, single-schema CMS for Astro with a typed local API and auto-generated admin UI.
---

# Kide CMS

Kide CMS is a code-first CMS for [Astro](https://astro.build). You define collections in TypeScript, and Kide generates the database tables, a typed content API, and an admin UI from that single schema — no separate admin app to maintain, no drift between your types and your data.

> This site is a **generated documentation preview** built by Docsbook from the [mhernesniemi/kide-cms](https://github.com/mhernesniemi/kide-cms) README and the project's own docs at [docs.kide.dev](https://docs.kide.dev/). It is a draft for a partner pitch, not the project's official documentation — always check [docs.kide.dev](https://docs.kide.dev/) for the current, authoritative source.

## Why one schema

Most headless CMS setups keep the content model in two places: a schema the admin UI understands, and types your app code trusts. Kide collapses that into one `defineCollection` call. One config generates:

- Drizzle database tables
- TypeScript types
- Zod validators
- The runtime admin UI (no generated page files — add a field and it appears)

## Stack

Astro 7, React 19, Drizzle ORM, SQLite (local) / D1 (Cloudflare), Zod, Tiptap, shadcn/ui, Tailwind CSS v4.

## Explore the docs

<!-- widget:cards -->

## Start here

- [Getting Started](./getting-started.md) — Scaffold a project, pick a runtime mode and a deploy target {rocket}
- [Collections](./guides/collections.md) — Define your content schema and get tables, types, and an admin UI for free {database}
- [Fields](./guides/fields.md) — Every field type, from plain text to drag-and-drop page blocks {shapes}

## Build and operate

- [Local API](./guides/local-api.md) — Query and mutate content from server code with full type safety {plug}
- [Access Control](./guides/access-control.md) — Role-based rules per collection and per field {shield}
- [Admin UI](./guides/admin-ui.md) — Customize list columns, live preview, and custom field components {layout-dashboard}
- [Deploy](./guides/deploy.md) — Ship to Node.js or Cloudflare Workers {cloud-upload}

<!-- /widget -->

## Try it

The fastest way to see Kide CMS is the project's own hosted demo, or scaffolding a project yourself:

```bash
pnpx create-kide-app
```

<!-- widget:cta -->

**See it running**

## Try the live Kide CMS admin

The maintainer runs a hosted instance of the generated admin UI so you can click through it before installing anything.

[Try live demo](https://demo.kide.dev/admin) · [View source on GitHub](https://github.com/mhernesniemi/kide-cms)

<!-- /widget -->
