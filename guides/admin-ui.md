---
title: Customizing the Kide CMS Admin UI
description: Configure list columns, field groups, live preview, custom field components, and navigation in the auto-generated Kide CMS admin interface.
---

# Admin UI

The admin is runtime-rendered from your schema — there are no generated page files. Add a field and it appears immediately.

## List columns

Configure which columns appear in the list view via `views` in the collection definition:

```ts
defineCollection({
  slug: "posts",
  views: {
    list: {
      columns: ["title", "category", "_status", "_updatedAt"],
      defaultSort: { field: "_updatedAt", direction: "desc" },
    },
  },
  fields: { /* ... */ },
});
```

List views page at a fixed 10 items — the page size is not configurable.

## Field position

Fields go to the content area by default. Set `admin.position: "sidebar"` to place a field in the sidebar:

```ts
fields: {
  title: fields.text({ required: true }), // → content
  body: fields.richText(),                // → content
  slug: fields.slug({ admin: { position: "sidebar" } }),     // → sidebar
  category: fields.text({ admin: { position: "sidebar" } }), // → sidebar
}
```

| Position | Description |
| --- | --- |
| `"content"` | Main area (left column on desktop), default |
| `"sidebar"` | Side panel (right column on desktop) |

## Field groups

Set `admin.group` to render fields inside a titled panel in the edit form. Consecutive fields sharing the same group become one panel; fields without a group render loose, as before. Grouping is purely presentational — field order and storage are unchanged.

```ts
fields: {
  heroHeading: fields.text({ label: "Heading", admin: { group: "Hero" } }),
  statValue: fields.text({ label: "Value", admin: { group: "Stats" } }),
  statLabel: fields.text({ label: "Label", admin: { group: "Stats" } }),
  notes: fields.text(), // ungrouped
}
```

The object form makes a panel collapsible: `collapsible: true` starts open, `"collapsed"` starts closed. Once an editor toggles a panel, the browser remembers that state per collection and group (localStorage), overriding the schema default on later visits.

## Preview

Collections with `pathPrefix` get a Preview link automatically. Collections without a prefix need `preview: true`. Singletons set `preview` to the URL directly:

```ts
// Automatic: pathPrefix enables preview
defineCollection({ slug: "posts", pathPrefix: "blog", /* ... */ });

// Explicit: no pathPrefix, needs opt-in
defineCollection({ slug: "pages", preview: true, /* ... */ });

// Singleton: set the URL directly
defineCollection({ slug: "front-page", singleton: true, preview: "/", /* ... */ });
```

The Preview link opens the public page in a new tab with `?preview=true`, which shows draft content.

### Live preview

When the preview tab is open, changes in the admin form update the preview in real time, no saving required — via `BroadcastChannel` same-origin messaging between tabs. Text fields (title, excerpt, etc.) update instantly via `textContent`; rich text and blocks render server-side via `/api/cms/preview/render` and get injected as HTML.

Enable live preview on a field by adding a `data-cms` attribute to the element that renders it, matching the field name from your collection definition:

```astro
<h1 data-cms="title">{doc.title}</h1>
<p data-cms="excerpt">{doc.excerpt}</p>
<div data-cms="body"><RichTextContent content={doc.body} /></div>
<div data-cms="blocks"><BlockRenderer blocks={blocks} /></div>
```

Only fields with `data-cms` attributes are live-updated; everything else updates on save, and the preview tab auto-reloads after saving. The client preview script is auto-injected on every page by the integration and only activates when `?preview` is in the URL, so there is zero overhead on normal page views.

### Public page setup

Check for the `?preview` param and query with `status: "any"` in preview mode — see [Local API](./local-api.md) for the full querying pattern:

```astro
---
import { cms } from "@/cms/.generated/api";

const isPreview = Astro.url.searchParams.has("preview");
const doc = await cms.posts.findOne({ slug: Astro.params.slug!, status: isPreview ? "any" : "published" });
if (!doc) return Astro.redirect("/404");
---
<h1 data-cms="title">{doc.title}</h1>
```

## Custom field components

Create a React component in `src/cms/fields/` and reference it by name:

```ts
// In your collection definition
color: fields.text({
  admin: { component: "ColorPicker" },
});
```

```tsx
// src/cms/fields/ColorPicker.tsx
import type { CustomFieldProps } from "@kidecms/core";

export default function ColorPicker({ name, value, readOnly }: CustomFieldProps) {
  return (
    <input type="color" name={name} defaultValue={value || "#000000"} disabled={readOnly} />
  );
}
```

The component receives `name` (form field name), `field` (field config), `value` (serialized value), and `readOnly`. It renders with `client:load` and must include an input with the `name` prop so the form can read its value.

Built-in component variants: `"radio"` (select as radio buttons), `"taxonomy-select"`, `"repeater"` (JSON array editor), `"color"` (palette picker), `"link"` (structured link), `"menu-items"`, `"taxonomy-terms"`.

## Custom navigation

Add custom pages to the admin sidebar via `admin.nav` in your CMS config:

```ts
export default defineConfig({
  admin: {
    nav: [
      { label: "Dashboard", href: "/dashboard", icon: "Home", weight: 10 },
      { label: "Analytics", href: "/analytics", icon: "BarChart", weight: 20 },
      { label: "Settings", href: "/settings", icon: "Settings" },
    ],
  },
  collections: [ /* ... */ ],
});
```

The sidebar sorts by group first, then by `weight` within each group. Groups render in a fixed order:

| Order | Group | Contents |
| --- | --- | --- |
| 0 | Content | Content collections |
| 10 | Library | Assets and related library items |
| 20 | Team | Users and team management |
| 100 | Custom | Your `admin.nav` items |

Custom nav items always land in the **Custom** group, which renders after every built-in group regardless of `weight` — `weight` only orders your items relative to each other within that group.

Collections control their own sidebar placement through collection-level `admin` config — see [Collections → Admin sidebar](./collections.md#admin-sidebar).

## Admin config

Configure admin behavior in your CMS config:

```ts
export default defineConfig({
  admin: {
    uploads: {
      allowedTypes: ["image/jpeg", "image/png", "image/webp", "application/pdf", "application/zip"],
      maxFileSize: 100 * 1024 * 1024, // 100 MB
    },
    rateLimit: {
      maxAttempts: 10,
      windowMs: 5 * 60 * 1000, // 5 minutes
    },
  },
  collections: [ /* ... */ ],
});
```

### Uploads

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `allowedTypes` | `string[]` | Images, PDF, MP4, WebM | Allowed MIME types |
| `maxFileSize` | `number` | `52428800` (50 MB) | Max file size in bytes |

SVG is deliberately not allowed by default — it executes script when served inline from the admin's origin. Re-enable it via `allowedTypes` only behind a CSP or `Content-Disposition`.

### Rate limiting

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `maxAttempts` | `number` | `5` | Login attempts before blocking |
| `windowMs` | `number` | `900000` | Time window in ms (default 15 min) |

### Colors

Define the palette offered by every `fields.color(...)` picker — editors choose from these named colors, there is no free-form hex entry:

```ts
admin: {
  colors: [
    { label: "Blue", value: "#4000FF" },
    { label: "Pink", value: "#FFDBEB" },
    { label: "Black", value: "#000000" },
  ],
}
```

A color field can override this list per-field via `fields.color({ colors: [...] })`.

### Date & time

| Option | Type | Default | Description |
| --- | --- | --- | --- |
| `dateFormat` | `string` | `"en-US"` | BCP-47 locale for date/time display (e.g. `"en-GB"`, `"fi-FI"`) |
| `timeZone` | `string` | browser | IANA time zone (e.g. `"Europe/Helsinki"`). Overrides each viewer's zone |
| `dateTimeFormat` | `Intl.DateTimeFormatOptions` | — | Overrides merged over the defaults |
| `dateTimePattern` | `string` | — | Explicit token pattern; wins over `dateFormat`/`dateTimeFormat` |

```ts
admin: {
  dateFormat: "fi-FI",
  timeZone: "Europe/Helsinki",
  dateTimePattern: "d.M.yyyy HH:mm", // → 1.7.2026 14:30
}
```

`dateFormat` is a locale, not a pattern — it controls ordering and 12/24-hour conventions. For finer control, use `dateTimeFormat` (e.g. `{ hour12: false }` for 24-hour time) or `dateTimePattern` for an exact layout. All settings are optional; defaults apply when omitted.
