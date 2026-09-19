---
title: Chapter 1 — Bootstrap the schema
description: Use schema.propose_migration to create your first tables. Auto-CRUD synthesises the default ObjectType + create/read/update/delete/list actions from the table shape, so one migration gives you a working data model on every surface.
---

# Chapter 1 — Bootstrap the schema (`schema.propose_migration`)

A Canonify app starts with tables. You declare a table with one
`schema.propose_migration` call, and **auto-CRUD** does the rest: from the
table shape the platform synthesises a default ObjectType and the five
CRUD ActionTypes — `<table>.create`, `<table>.read`, `<table>.update`,
`<table>.delete`, `<table>.list` — all live on REST, CLI, and MCP the
moment the migration commits.

> Example domain note: this chapter uses `customer` / `contact` / `deal`
> as an illustrative CRM. They are examples, not product commitments —
> substitute your own nouns.

## The action

| | |
|---|---|
| **Action** | `schema.propose_migration` |
| **Input** | `{ description: string, sql: string }` |
| **Envelope** | `{ data: { migration_id, ... }, audit_event_id, approval_status: 'not_required' }` |
| **Idempotency** | re-running an identical migration is safe |

`description` is a human sentence that lands in the audit trail; `sql` is a
single `CREATE TABLE` (or `ALTER TABLE`) statement.

## Conventions (from `AGENTS.md`)

- `id TEXT PRIMARY KEY` — holds a **prefixed ULID** (`cust_…`, `cnt_…`,
  `dl_…`). The prefix is derived from the ObjectType and stamped by the
  runtime on insert; you never generate ids yourself.
- `created_at` / `updated_at` are `TEXT NOT NULL` ISO 8601 UTC strings —
  **never** `INTEGER` epoch.
- A column you intend to drive with a StateMachine later (Chapter 3) is a
  plain `TEXT NOT NULL DEFAULT '<initial>'` here; you narrow it to an enum
  in Chapter 2.
- Tables prefixed with `_` (`_audit_event`, `_outbox`, `_view_specs`, …)
  are platform-owned. Do not declare them.

## The three example tables

```sql
CREATE TABLE customer (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'prospect',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE contact (
  id          TEXT PRIMARY KEY,
  customer_id TEXT NOT NULL,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  role        TEXT,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE deal (
  id          TEXT PRIMARY KEY,
  customer_id TEXT NOT NULL,
  title       TEXT NOT NULL,
  amount      TEXT NOT NULL,
  stage       TEXT NOT NULL DEFAULT 'open',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);
```

Each `CREATE TABLE` is its own `schema.propose_migration` call.

## Invoke it — CLI

```sh
canon actions invoke schema.propose_migration --input - <<'JSON'
{
  "description": "customer table — the CRM root entity",
  "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'prospect', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
}
JSON
```

You can also point at a file: `--input @customer-migration.json`. Output
is text by default; pass `--output json` when you are an agent parsing the
envelope.

## Invoke it — REST

```sh
curl -X POST https://api.canonify.app/v1/actions/schema.propose_migration \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "description": "customer table — the CRM root entity",
      "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT '\''prospect'\'', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
    }
  }'
```

The body is always `{ input, idempotency_key? }`. For local testing you
can pass `X-Principal-Json` instead of a bearer token.

## Invoke it — MCP

Call the `actions.invoke` tool with `{ name, args }`:

```json
{
  "name": "actions.invoke",
  "arguments": {
    "name": "schema.propose_migration",
    "args": {
      "description": "customer table — the CRM root entity",
      "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'prospect', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
    }
  }
}
```

## Confirm auto-CRUD fired

```sh
canon actions list --json | jq '.[].name' | grep '^customer\.'
# customer.create
# customer.read
# customer.update
# customer.delete
# customer.list

canon objects list-types | grep customer
canon schema describe --object customer
```

You now have a working, queryable data model. The default ObjectType is
permissive — every column is a `mapped` `string`, no links — which is what
Chapter 2 fixes.

## Failure modes

| Symptom | Likely cause | Recovery |
|---|---|---|
| `VALIDATION` on the SQL | unsupported DDL or a `_`-prefixed table name | use a single `CREATE TABLE` / `ALTER TABLE`; drop the `_` prefix |
| `customer.*` actions missing afterwards | migration failed / rolled back | read the envelope `error`; `canon audit <audit_event_id>` for detail |
| `INVALID_STATE` re-running | re-proposing a structurally different migration | adjust the SQL or run an `ALTER TABLE` migration instead |

## Next

→ [Chapter 2 — Refine the ObjectType](objecttype-refinement.md): narrow
`status`/`stage` to enums and link the three ObjectTypes together.
