---
title: Field Types Reference for Kide CMS
description: Reference for every Kide CMS field type, from text and relations to drag-and-drop blocks, plus common options and field-level access rules.
---

# Fields

## Field types

| Field | Storage | Admin component |
| --- | --- | --- |
| `text` | `text` | Input or Textarea (with `rows`) |
| `slug` | `text` (unique) | Auto-generated from source field |
| `email` | `text` | Email input |
| `number` | `integer` | Number input |
| `boolean` | `integer` (0/1) | Checkbox |
| `date` | `text` (ISO 8601) | Date picker |
| `select` | `text` | Select dropdown |
| `color` | `text` (hex) | Palette swatch dropdown |
| `link` | `text` (JSON) | Internal/external link control |
| `richText` | `text` (JSON AST) | Tiptap editor |
| `content` | `text` (JSON AST) | Tiptap editor with inline blocks |
| `image` | `text` | Image picker with upload/browse |
| `relation` | `text` (reference ID) | Combobox with search |
| `array` | `text` (JSON) | Comma-separated input (`of: fields.text()`), else JSON textarea |
| `json` | `text` (JSON) | Textarea or custom component |
| `blocks` | `text` (JSON) | Drag-and-drop block editor |

## Common options

All fields accept these:

| Option | Type | Description |
| --- | --- | --- |
| `required` | `boolean` | Validate as non-empty on save |
| `label` | `string` | Custom label (defaults to humanized field name) |
| `description` | `string` | Text shown below the label |
| `defaultValue` | varies | Initial value for new documents |
| `translatable` | `boolean` | Store per-locale in the translations table |
| `indexed` | `boolean` | Add a database index |
| `unique` | `boolean` | Enforce unique values (defaults to `true` for `slug`, `false` otherwise) |
| `condition` | `{ field, value }` | Show/hide based on another field |
| `admin.placeholder` | `string` | Input placeholder text |
| `admin.rows` | `number` | Textarea height (text fields) |
| `admin.help` | `string` | Help text below the input |
| `admin.position` | `"content" \| "sidebar"` | Where the field renders in the edit form. Defaults to `"content"` |
| `admin.group` | `string \| object` | Titled, optionally collapsible panel — consecutive fields sharing a group render together |
| `admin.hidden` | `boolean` | Hide from the admin UI |
| `admin.component` | `string` | Custom admin component |

## Type-specific options

| Field | Option | Type | Description |
| --- | --- | --- | --- |
| `text` | `maxLength` | `number` | Maximum character length, validated on save |
| `slug` | `from` | `string` | Field name the slug is auto-generated from |
| `select` | `options` | `string[]` (required) | Allowed values |
| `relation` | `collection` | `string` (required) | Target collection slug |
| `relation` | `hasMany` | `boolean` | Store an array of IDs instead of one |
| `relation` | `maxItems` | `number` | Max selected documents (`hasMany` only) |
| `array` | `of` | field (required) | Item field type, e.g. `fields.text()` |
| `array` | `maxItems` | `number` | Max items, enforced on save |
| `content` | `blocks` | `object` | Inline component block types |
| `content` | `fullscreen` | `boolean` | Distraction-free overlay button. Defaults to `true` |
| `blocks` | `types` | `object` (required) | Block types, each a map of sub-fields |
| `blocks` | `shared` | `boolean` | Allow shared section references. Defaults to `true` |

## Blocks

```ts
fields.blocks({
  types: {
    hero: {
      heading: fields.text({ required: true }),
      body: fields.text(),
      ctaLabel: fields.text(),
      ctaHref: fields.text(),
    },
    text: {
      heading: fields.text(),
      content: fields.richText(),
    },
    faq: {
      heading: fields.text(),
      items: fields.json({
        defaultValue: [],
        admin: { component: "repeater" },
      }),
    },
  },
});
```

Block sub-fields accept any field type — common choices are `text`, `number`, `boolean`, `select`, `richText`, `image`, `relation`, `array`, `json`, `color`.

The `repeater` component renders JSON arrays as sortable add/remove item cards. Declare `itemFields` to give rows typed controls instead of free-form JSON, including `link` sub-fields with the internal-link picker:

```ts
services: fields.json({
  admin: { component: "repeater" },
  itemFields: {
    title: fields.text({ required: true }),
    body: fields.text({ admin: { rows: 3 } }),
    link: fields.link(),
  },
}),
```

Repeaters work both as block sub-fields and as top-level collection fields — useful for fixed-slot templates where a section holds a list of cards.

## Content (prose + inline blocks)

`fields.content(...)` is for long-form editing where prose and inline component blocks live in one ordered stream, such as a blog post body:

```ts
body: fields.content({
  translatable: true,
  admin: { rows: 14 },
  fullscreen: true,
  blocks: {
    faq: {
      heading: fields.text(),
      items: fields.json({ admin: { component: "repeater" } }),
    },
    image: {
      images: fields.array({ of: fields.image(), defaultValue: [] }),
    },
  },
});
```

Each inline block uses a `blockType` and `fields` payload, similar in shape to `fields.blocks(...)`, but the block appears inside the writing flow rather than as a separate page-section list. Render content fields with `ContentRenderer`, which interleaves prose and the configured block components. Use `fields.richText(...)` when the editor only needs formatted text, and `fields.blocks(...)` when the editor should manage standalone, reusable page sections.

When `fullscreen` is enabled, a maximize button appears in the editor's top-right corner. It opens a distraction-free overlay that hides everything else and centers the writing column; **Escape** or **Exit** returns to the normal form with edits preserved.

## Item limits and ordering

List-shaped fields (`relation` with `hasMany`, `array`, and `json` repeaters) accept `maxItems` — enforced centrally on save and reflected in the admin picker with a `2/4 selected` counter:

```ts
newsItems: fields.relation({
  collection: "articles",
  hasMany: true,
  maxItems: 4,
}),
```

The order of a `hasMany` selection is data — stored and rendered as-is. The admin renders selections as drag-sortable rows (grip handle, `#index`, remove) by default, no option needed.

## Colors

`fields.color(...)` stores a hex string and renders a dropdown of predefined swatches — editors pick a named color instead of typing a hex value. Define the palette once in your CMS config and every color field offers it:

```ts
export default defineConfig({
  admin: {
    colors: [
      { label: "Blue", value: "#4000FF" },
      { label: "Pink", value: "#FFDBEB" },
      { label: "Black", value: "#000000" },
    ],
  },
  collections: [ /* ... */ ],
});
```

```ts
// Uses the global admin.colors palette
backgroundColor: fields.color(),
```

Override the palette for a single field with `colors` — it wins over the global list.

## Shared sections

Editors can save a `fields.blocks(...)` block as a shared section, insert it into other block fields, and detach it later into a local copy. Shared sections live in the admin sidebar under **Shared Sections**, stored as normal draft/versioned content — the right fit for reusable heroes, CTAs, FAQ sections, or feature grids.

Set `shared: false` for block fields that should never use shared sections, such as form builders. Shared sections can be inserted from the editor's `/` slash menu; a regular block can be saved as a shared section, and a shared block can be detached back into an editable copy.

## Conditional fields

Show or hide fields based on a `select` or `boolean` field's value:

```ts
postType: fields.select({
  options: ["article", "video", "podcast"],
}),
videoUrl: fields.text({
  condition: { field: "postType", value: "video" },
}),
```

`value` can be a string, boolean, or array of strings (matches any).

## Field-level access

Fields support `read` and `update` access rules:

```ts
import { hasRole } from "@kidecms/core";

summary: fields.text({
  access: {
    read: hasRole("admin"), // hidden from non-admins
  },
}),
seoDescription: fields.text({
  access: {
    update: hasRole("admin"), // read-only for non-admins
  },
}),
```

| Rule | Effect in admin UI | Effect on save |
| --- | --- | --- |
| `read` | Field is completely hidden | Field excluded from response |
| `update` | Field is rendered read-only (disabled) | Field value silently preserved (changes stripped) |

Both rules receive the same context as collection-level access rules — see [Access Control](./access-control.md).

## AI assistant

The admin includes optional AI features powered by the Vercel AI SDK. Add `AI_PROVIDER=openai` and `AI_API_KEY` to `.env` to enable them; `AI_MODEL` is optional and defaults to `gpt-4o-mini`. Currently `openai` is the only supported provider.

When configured, AI buttons appear automatically for:

- **Alt text generation** on asset detail pages
- **SEO descriptions** on post/page edit forms
- **Translation**, with per-field "Translate from EN" buttons that handle both plain text and rich text (preserving JSON AST structure)

Without the AI env vars, all AI buttons are hidden and no AI dependencies are loaded.
