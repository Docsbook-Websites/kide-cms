---
title: Querying Content with the Local API
description: Use the typed cms.<collection> API to find, create, update, publish, and translate documents directly from Astro server code, no HTTP required.
---

# Local API

Import the generated API and call it directly, without HTTP overhead. Every query and return type is fully typed based on your collection definitions.

```ts
import { cms } from "@/cms/.generated/api";
```

Every collection is available as `cms.<slug>`. For example, with collections `posts`, `pages`, and `users`:

```ts
cms.posts.find({ ... })
cms.pages.findOne({ slug: "about" })
cms.users.findById("abc123")
```

## Find

```ts
const posts = await cms.posts.find({
  where: { category: "tech" },
  sort: { field: "_updatedAt", direction: "desc" },
  limit: 10,
  offset: 0,
  status: "published",
  locale: "fi",
});
```

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `where` | `Record<string, unknown>` | — | Filter by field values |
| `sort` | `{ field, direction }` | — | Sort by field, `"asc"` or `"desc"` |
| `limit` | `number` | — | Max documents to return |
| `offset` | `number` | — | Skip N documents |
| `status` | `"draft" \| "published" \| "scheduled" \| "any"` | `"published"` (draft-enabled) / `"any"` (others) | Filter by status |
| `locale` | `string` | default locale | Language code for translations |
| `search` | `string` | — | Substring search across `text`, `slug`, `email`, and `select` fields |

## Find one and find by ID

```ts
const post = await cms.posts.findOne({
  slug: "hello-world",
  locale: "fi",
  status: "any",
});

const same = await cms.posts.findById("abc123", {
  locale: "fi",
  status: "any",
});
```

`findById` filters by status only when `status` is passed explicitly — a value other than `"any"` returns `null` unless the document has that status. By default the document is returned regardless of status. For draft-enabled collections, the default (and an explicit `"published"`) also overlays the last-published snapshot values on the result; pass `status: "any"` to read the current working values without filtering.

## Create, update, delete

```ts
const post = await cms.posts.create({
  title: "New Post",
  body: { type: "root", children: [ /* ... */ ] },
  _status: "published", // optional, defaults to "draft" if drafts enabled
});

await cms.posts.update("abc123", { title: "Updated Title" });

await cms.posts.delete("abc123");
```

- **Create** accepts field values as properties and returns the created document. You can supply your own `_id` to make imports idempotent; otherwise one is generated.
- **Create many** — `cms.posts.createMany([{ title: "A" }, { title: "B" }])` runs documents through `create` sequentially.
- **Upsert** — `cms.posts.upsert({ _id: "abc123", title: "Hello" })` updates when a document with that `_id` exists, otherwise creates it. Combined with caller-supplied `_id`s, this makes imports re-runnable without a wipe-first step.
- **Update** only needs the fields you want to change; it returns the updated document.
- **Delete** cascades — removes translations, versions, and the document. Returns `true` if a row was removed, `false` if not found. Auth collections refuse to delete the last remaining admin.
- **Delete many** — `cms.posts.deleteMany({ category: "tech" })` deletes every matching document (all documents if the filter is omitted) and returns the count removed.

## Publish, unpublish, discard, schedule

```ts
await cms.posts.publish("abc123");
await cms.posts.unpublish("abc123");
await cms.posts.discardDraft("abc123"); // revert pending changes to last-published content

await cms.posts.schedule(
  "abc123",
  "2025-06-01T00:00:00Z", // publishAt (required)
  "2025-07-01T00:00:00Z", // unpublishAt (optional)
);
```

`schedule` sets status to `"scheduled"`. On Cloudflare, a cron trigger processes scheduled publishing automatically. On Node.js, set up an external cron to call `/api/cms/cron/publish` and secure it with the `CRON_SECRET` env var — see [Deploy](./deploy.md).

`discardDraft` only works on published documents that have pending edits.

## Count, versions, translations

```ts
const total = await cms.posts.count({ status: "published" });

const versions = await cms.posts.versions("abc123");
await cms.posts.restore("abc123", 5);

const translations = await cms.posts.getTranslations("abc123");
await cms.posts.upsertTranslation("abc123", "fi", {
  title: "Hei maailma",
  body: { type: "root", children: [ /* ... */ ] },
});
```

`upsertTranslation` inserts or updates and only needs translatable fields.

## Access context

Every method accepts an optional context object as its last argument. Pass the signed-in admin user to enforce [field-level access](./access-control.md), or `_system: true` to bypass all access rules in trusted server code such as public API routes, task handlers, and seed scripts:

```ts
await cms.posts.find({}, { user });
await cms["form-submissions"].create(data, { _system: true });
```

## Introspection

```ts
cms.meta.getCollections();                    // All collections with metadata
cms.meta.getFields("posts");                  // Field definitions for a collection
cms.meta.getCollection("posts");              // Full collection config
cms.meta.getRouteForDocument("posts", doc);   // Public URL for a document
cms.meta.getLocales();                        // { default, supported }
cms.meta.isTranslatableField("posts", "title"); // true/false
cms.meta.getConfig();                         // Full CMS config object
```

Background task helpers also exist as `cms.tasks.enqueue`, `cms.tasks.drain`, `cms.tasks.tick`, and `cms.tasks.prune`.

## Usage in Astro pages

```astro
---
import { cms } from "@/cms/.generated/api";

const post = await cms.posts.findOne({ slug: Astro.params.slug });
if (!post) return Astro.redirect("/404");

Astro.cache.set({ tags: ["posts", `post:${post._id}`] });
---
<h1>{post.title}</h1>
```

The raw tag array shown above is equivalent to the `cacheTags()` helper.
