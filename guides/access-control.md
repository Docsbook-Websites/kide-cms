---
title: Access Control and Roles in Kide CMS
description: Define role-based access rules per collection and per field in Kide CMS, and understand the default-deny protections built into auth collections.
---

# Access Control

Access rules are defined directly in each collection config file using helper functions. Rules are pure functions evaluated before every API operation. Any operation without a rule is allowed by default — except on `auth` collections, which are default-deny (see below).

```ts
import { defineCollection, fields, hasRole } from "@kidecms/core";

export default defineCollection({
  slug: "posts",
  access: {
    create: hasRole("admin", "editor"),
    update: hasRole("admin", "editor"),
    delete: hasRole("admin"),
    publish: hasRole("admin", "editor"),
  },
  fields: { /* ... */ },
});
```

In this example, `read` is not set, so anyone can read posts. Only the operations you want to restrict need a rule.

## Auth collections

A collection with `auth: true` holds the passwords and roles that every other rule is checked against, so default-allow would let any signed-in account promote itself to `admin` or overwrite someone else's password. These collections are therefore **default-deny**:

| Operation | Without an explicit rule |
| --- | --- |
| `create`, `delete` | Admins only |
| `read`, `update` | Admins, or your own record |
| `publish`, `schedule` | Admins only |

Two fields are protected regardless of the collection rule, so loosening `access.update` can't accidentally re-open privilege escalation:

- **`role`** — only an admin can set it, including on their own record.
- **`password`** — an admin, or the owner of that record. This is what lets a non-admin change their own password.

Create and update differ: on **create**, only `role` is stripped for non-admins, so a new account can still set its initial password. On **update**, both `role` and `password` are stripped for non-admins, unless a field-level `access.update` rule overrides it.

Declaring your own rule replaces the default for that operation, and a field-level `access.update` rule on `role` or `password` replaces the field protection. Server-side code can still bypass everything with `{ _system: true }`.

## Helper functions

| Helper | Description |
| --- | --- |
| `hasRole("admin")` | Checks if the user has the specified role |
| `hasRole("admin", "editor")` | Accepts multiple roles (OR logic) |

## Operations

| Operation | When checked |
| --- | --- |
| `read` | `find`, `findOne`, `findById`, `count` |
| `create` | `create` |
| `update` | `update` |
| `delete` | `delete` |
| `publish` | `publish`, `unpublish` |
| `schedule` | `schedule` (falls back to `publish` rule) |

## Context

Rules receive `({ user, doc, operation, collection }) => boolean | Promise<boolean>` and can be async — return a promise and the API awaits it.

- `user` — current session user (`{ id, role, email, ...customUserFields }` or `null`)
- `doc` — existing document. For `find` and `count`, `read` rules are evaluated per document, so rules can hide individual rows.
- `operation` — operation name
- `collection` — collection slug

## Field-level access

Individual fields support `read` and `update` access rules:

```ts
summary: fields.text({
  access: {
    read: hasRole("admin"),   // hidden from non-admins
    update: hasRole("admin"), // read-only for non-admins
  },
}),
```

Both rules are enforced in the API, not just the admin UI:

- `read` — the field is omitted entirely from API responses when the rule denies, and hidden from the admin UI.
- `update` — denied values are stripped from create and update payloads before they reach the database; in the admin UI the field renders read-only and the existing value is preserved on save.

## Audit log

Kide writes an audit trail of admin actions to the `cms_audit_log` table, covering content, asset, user, invite, session, and password-reset events. Each entry records the action, the actor's id, email, and role (or the attempted email on a failed login), and the request's IP address and user agent. Every entry is also emitted as a structured log line (`kind: "audit"`) so it reaches stdout-based log pipelines.

The activity feed in collaboration-enabled collections reads directly from this log. The cron tasks endpoint prunes entries older than 90 days on each run; call `pruneAuditLog(olderThanMs)` from `@kidecms/core` yourself for a different retention window.
