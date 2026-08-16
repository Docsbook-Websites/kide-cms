---
title: Defining Collections in Kide CMS
description: Learn how Kide CMS collections become database tables, typed APIs, and admin screens, and how to add drafts, versions, singletons, and locales.
---

# Collections

Collections are defined in `src/cms/collections/` and registered in `src/cms/cms.config.ts`. Each collection becomes a database table, a set of TypeScript types, Zod validators, and an admin UI — generated from one config, not maintained separately.

## Basic collection

```ts
// src/cms/collections/posts.ts
import { defineCollection, fields, hasRole } from "@kidecms/core";

export default defineCollection({
  slug: "posts",
  labels: { singular: "Post", plural: "Posts" },
  timestamps: true,
  drafts: true,
  versions: { max: 20 },
  access: {
    publish: hasRole("admin"),
  },
  fields: {
    title: fields.text({ required: true }),
    slug: fields.slug({
      from: "title",
      admin: { position: "sidebar" },
    }),
    body: fields.richText(),
    author: fields.relation({
      collection: "authors",
      admin: { position: "sidebar" },
    }),
  },
});
```

Register it in `src/cms/cms.config.ts`:

```ts
import { defineConfig } from "@kidecms/core";
import posts from "./collections/posts";

export default defineConfig({
  database: { dialect: "sqlite" },
  collections: [posts],
});
```

## Collection options

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `slug` | `string` | — | URL-safe identifier, used as table name prefix |
| `labels` | `{ singular, plural }` | — | Display names in admin |
| `singleton` | `boolean` | `false` | Single document (e.g. front page) |
| `timestamps` | `boolean` | `true` | Auto `_createdAt` / `_updatedAt` |
| `drafts` | `boolean` | `false` | Enable draft/published status |
| `versions` | `{ max: number }` | — | Keep version snapshots |
| `pathPrefix` | `string` | — | URL prefix for public pages (e.g. `"blog"`) |
| `preview` | `boolean \| string` | `false` | Enable preview link (string for static URL) |
| `labelField` | `string` | `title` | Field used as document display name |
| `access` | `object` | — | Role-based access rules — see [Access Control](./access-control.md) |
| `hooks` | `object` | — | Lifecycle hooks |
| `views` | `object` | — | List column config — see [Admin UI](./admin-ui.md) |
| `admin` | `object` | — | Sidebar grouping, icon, order, and visibility |
| `auth` | `boolean` | `false` | Mark an authentication-sensitive collection |
| `searchable` | `boolean \| { fields: string[] }` | — | Controls full-text search inclusion |

Seed content is not a collection option — define it in `src/cms/seed.ts`, keyed by collection slug, and load it with `pnpm cms:seed`.

The built-in login flow uses a `users` collection marked with `auth: true`. Auth collections get stricter default access rules, automatic password hashing, and credential-field filtering — see [Access Control → Auth collections](./access-control.md).

## Admin sidebar

Collection-level `admin` options keep large projects organized:

```ts
defineCollection({
  slug: "case-studies",
  labels: { singular: "Case Study", plural: "Case Studies" },
  admin: {
    group: "Marketing",
    icon: "Star",
    weight: 20,
  },
  fields: { /* ... */ },
});
```

| Option | Type | Description |
| --- | --- | --- |
| `group` | `string` | Sidebar group label. Built-ins include `Content`, `Library`, `Team` |
| `icon` | `string` | Lucide icon name used in the sidebar |
| `weight` | `number` | Sort order inside the group — lower appears first |
| `sidebar` | `boolean` | Set `false` to hide from the sidebar |

Custom groups are collapsible and remember their open/closed state in the browser. Hidden collections still work in relations, search, and the local API.

## Label field

By default, the admin uses a collection's `title` field as the document display name (relation selects, list view, breadcrumbs). If your collection has no `title` field, or you want a different one, set `labelField`:

```ts
defineCollection({
  slug: "authors",
  labels: { singular: "Author", plural: "Authors" },
  labelField: "name",
  fields: {
    name: fields.text({ required: true }),
    title: fields.text(), // work title, not the display name
  },
});
```

Fallback chain: `labelField` → field named `title` → field named `name` → first text field → first field of any type.

## Singletons

```ts
defineCollection({
  slug: "front-page",
  labels: { singular: "Front Page", plural: "Front Page" },
  singleton: true,
  fields: { /* ... */ },
});
```

Singletons show under "Singles" in the sidebar. One document per collection.

## Internationalization

Enable locales in `cms.config.ts` and mark fields as `translatable`:

```ts
// src/cms/cms.config.ts
export default defineConfig({
  locales: {
    default: "en",
    supported: ["en", "fi"],
  },
  collections: [posts],
});
```

```ts
// src/cms/collections/posts.ts
fields: {
  title: fields.text({ translatable: true }),
  body: fields.richText({ translatable: true }),
  category: fields.select({ options: ["tech", "design"] }), // not translated
}
```

Translatable fields get a separate `_translations` table; non-translatable fields stay on the main table. The admin shows a language switcher on edit pages.

## Next steps

<!-- widget:cards -->

- [Fields](./fields.md) — Full reference for every field type used above {shapes}
- [Local API](./local-api.md) — Query and mutate the collections you just defined {plug}
- [Access Control](./access-control.md) — Restrict who can create, edit, or publish {shield}

<!-- /widget -->
