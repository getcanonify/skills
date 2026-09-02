# Declaring tables — `schema.propose_migration`

Tables are how data lives in Canonify. Declaring a table is step one of
building a custom app. The platform Action you invoke is
`schema.propose_migration`; it applies the SQL in a transaction
(rolling back on failure) and then runs `registerAutoCrud` so the new
table immediately gets a default ObjectType plus the five ordinary CRUD
Actions (`<table>.create`, `<table>.read`, `<table>.update`,
`<table>.delete`, `<table>.list`). A compound root additionally gets
`<table>.replace` when a `part_of` child link is present; that compound action
can be disabled with `defaults.replace: false`.

After the migration commits, the new actions are discoverable from any
surface without a restart:

```sh
canon actions list --json | jq '.[] | select(.name | startswith("lead."))'
```

See [Declaring ObjectTypes](declaring-objecttypes.md) for the
refinement story that layers on top of the default ObjectType and the
override hierarchy.

---

## Required column conventions

From `AGENTS.md`:

- **Primary key**: `id TEXT PRIMARY KEY`. The runtime generates
  prefixed ULIDs (`cust_…`, `ld_…`) on insert; the prefix is derived
  from the table name unless you override it via `catalog.refine_object_type`.
- **Timestamps**: `created_at TEXT NOT NULL` and `updated_at TEXT NOT NULL`
  — ISO 8601 UTC strings, **never** `INTEGER` epoch. The executor
  stamps both on `insert`; `update` steps refresh `updated_at`
  automatically when the property exists.
- **System tables** start with `_` (`_audit_event`, `_outbox`,
  `_action_types`, etc.). Don't create new ones; the platform owns the
  underscore namespace.
- **Snake_case** column names; **camelCase** TypeScript properties
  downstream.

---

## Invocation shape

```sh
canon actions invoke schema.propose_migration --input @lead-table.json
```

`lead-table.json`:

```json
{
  "description": "lead table — pre-conversion sales prospect",
  "sql": "CREATE TABLE lead (\n  id            TEXT PRIMARY KEY,\n  name          TEXT NOT NULL,\n  email         TEXT NOT NULL,\n  source        TEXT NOT NULL,\n  status        TEXT NOT NULL DEFAULT 'new',\n  contacted_at  TEXT,\n  qualified_at  TEXT,\n  converted_at  TEXT,\n  customer_id   TEXT,\n  created_at    TEXT NOT NULL,\n  updated_at    TEXT NOT NULL\n)"
}
```

Or via the structured `actions invoke` form (literal YAML, as used by
the worked scenarios in the `canonify-scenarios` skill):

```yaml
action: schema.propose_migration
input:
  description: lead table — pre-conversion sales prospect
  sql: |
    CREATE TABLE lead (
      id            TEXT PRIMARY KEY,
      name          TEXT NOT NULL,
      email         TEXT NOT NULL,
      source        TEXT NOT NULL,
      status        TEXT NOT NULL DEFAULT 'new',
      contacted_at  TEXT,
      qualified_at  TEXT,
      converted_at  TEXT,
      customer_id   TEXT,
      created_at    TEXT NOT NULL,
      updated_at    TEXT NOT NULL
    )
```

Envelope on success:

```json
{
  "data": { "applied": true, "table": "lead" },
  "audit_event_id": "aud_…"
}
```

---

## What auto-CRUD synthesises

After the migration commits, the registry now contains:

- ObjectType `lead` with one `mapped` property per column, typed by the
  SQL column type (TEXT → `string`, INTEGER → `number`, etc.). All
  properties are permissive (string, never enum) — narrow them via
  [`catalog.refine_object_type`](declaring-objecttypes.md).
- Five ordinary Action types: `lead.create`, `lead.read`, `lead.update`,
  `lead.delete`, `lead.list`. Each schema-validates its input against
  the inferred column types and writes through `runtime.invoke` with
  full audit + idempotency + outbox semantics.
- A compound root with a `part_of` child link also gets `lead.replace`, which
  reconciles the whole root-and-parts cluster atomically. Set
  `defaults.replace: false` during refinement to suppress that action; on a
  type with no `part_of` children, the same flag is accepted as a harmless
  no-op because `replace` was never generated.

The defaults can be disabled per verb via the `defaults` field on
`catalog.refine_object_type` (`{ delete: false }` makes the row
undeleteable — the Action ceases to exist on every surface; use
`{ replace: false }` for a compound root).

---

## Multiple tables in one session

`schema.propose_migration` is one table per call. For a multi-table
schema, dispatch a sequence:

```yaml
- action: schema.propose_migration
  input: { description: lead table,    sql: "CREATE TABLE lead (…)" }
- action: schema.propose_migration
  input: { description: contact table, sql: "CREATE TABLE contact (…)" }
- action: schema.propose_migration
  input: { description: deal table,    sql: "CREATE TABLE deal (…)" }
- action: schema.propose_migration
  input: { description: note table,    sql: "CREATE TABLE note (…)" }
```

Each call is independently atomic — a failure on table 2 leaves table 1
committed. Idempotency: re-applying a `CREATE TABLE` for an existing
table fails the SQL and surfaces as `VALIDATION`. The handler is
`idempotency: 'per-key'` so retries with the same idempotency key
return the cached envelope.

---

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| Bad SQL (`CRATE TABLE …`) | `VALIDATION` | Fix the SQL and retry; the migration was rolled back. |
| Table name already exists | `VALIDATION` | Either pick a new name or migrate forward (`ALTER TABLE …`). |
| Missing required column (`id` / `created_at` / `updated_at`) | `VALIDATION` | Add the columns; auto-CRUD depends on them. |
| Underscore-prefixed name (`_my_table`) | `VALIDATION` | The platform owns `_*`; pick a non-underscore name. |

See [`error-codes.md`](error-codes.md) for the full kind table.

---

## See also

- [Declaring ObjectTypes](declaring-objecttypes.md) — refine the default ObjectType auto-CRUD generates
- [Declaring StateMachines](declaring-state-machines.md) — bind an FSM to an enum column
- [Declaring custom ActionTypes](declaring-actions.md) — write a handler that spans multiple tables
- Phase 17.5 §7 (platform spec) — refinement contract
