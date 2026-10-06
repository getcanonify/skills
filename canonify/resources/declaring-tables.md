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
  `id` must be unique: the single-column PRIMARY KEY (or `id TEXT NOT NULL
  UNIQUE`). A migration that leaves a table with a non-unique `id` — a plain
  `id TEXT`, a composite key, a generated `id`, a unique index looser than
  the column's collation, `CREATE TABLE … AS SELECT id …` — is refused with
  `details.code: 'id_not_unique'`. A table declaring `ON CONFLICT REPLACE`
  on any column is refused with `details.code: 'on_conflict_replace'`, and
  the `id` property cannot be re-pointed at another column by
  `catalog.refine_object_type`. The check covers every table, not only the
  one a migration touches: a table already in either shape blocks every
  other migration until it is repaired (`details.preexisting` names it).
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

## Changing a column on a table with rows — `column_modify`

SQLite has no `ALTER COLUMN`. To change one column's declared type or its
nullability, pass `column_modify` instead of `sql`. The platform rebuilds
the table in one call: it keeps every row, the table's own `CREATE TABLE`
text (only that column changes), its indexes, UNIQUE / CHECK / REFERENCES /
COLLATE clauses and triggers, and it rebuilds the search index. Other
tables' rows that reference it keep their links, and views and other
tables' triggers that name it are kept as written.

```yaml
- action: schema.propose_migration
  input:
    description: money in minor units
    column_modify: { table: invoice, column: amount, type: INTEGER, cast: round }
```

| Field | Meaning |
|---|---|
| `nullable` | `true` drops NOT NULL, `false` adds it. |
| `type` | New declared type, a type name only (`INTEGER`, `TEXT`, `NUMERIC(10, 2)`). |
| `cast` | Accepts a lossy conversion: `round` (to the nearest integer; INTEGER targets only) or `truncate` (SQLite's CAST: toward zero; for other targets, SQLite's own conversion). |

Give `nullable`, `type`, or both. Without `cast`, a type change is refused
(`details.code: 'lossy_conversion'`, with `rows` and `example_ids`) when any
stored value would change: `31.25` into INTEGER, `'007'` into INTEGER, or a
REAL that loses digits as TEXT. Whole numbers (`3120.0` → `3120`) convert
without one. Text that is not a number, a blob, or a number beyond the
64-bit range is refused even with a cast (`unconvertible_value`): fix or
clear those values first. The row key (PRIMARY KEY) cannot change type
(`pk_type_change`). Update your bundle's `CREATE TABLE` to match, and the
next apply reports no `schema_ddl_divergence`.

---

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| Bad SQL (`CRATE TABLE …`) | `VALIDATION` | Fix the SQL and retry; the migration was rolled back. |
| Table name already exists | `VALIDATION` | Either pick a new name or migrate forward (`ALTER TABLE …`). |
| Missing required column (`id` / `created_at` / `updated_at`) | `VALIDATION` | Add the columns; auto-CRUD depends on them. |
| Underscore-prefixed name (`_my_table`) | `VALIDATION` | The platform owns `_*`; pick a non-underscore name. |
| `id` not the PRIMARY KEY / UNIQUE (`details.code: 'id_not_unique'`) | `VALIDATION` | Declare `id TEXT PRIMARY KEY`; for an existing table, `CREATE UNIQUE INDEX "<t>_id_unique" ON "<t>" (id)`. |
| `ON CONFLICT REPLACE` clause (`details.code: 'on_conflict_replace'`) | `VALIDATION` | Drop the conflict clause; the default (ABORT) refuses a colliding write instead of deleting the other row. For an existing table, a `column_modify` migration rebuilds it without the clause and keeps its rows. |
| A type change would change stored values (`details.code: 'lossy_conversion'`) | `VALIDATION` | Nothing changed. Add `cast: round` or `cast: truncate` to accept the conversion, or fix the listed rows first. |
| A value cannot become the new type (`details.code: 'unconvertible_value'`) | `VALIDATION` | Nothing changed. Fix or clear the listed rows (text that is not a number, blobs), then retry. |
| A rebuild would break other rows' references (`details.code: 'foreign_key_violation'`, with `tables` and `rows`) | `VALIDATION` | Nothing changed. A type change made a referenced key stop matching its references (e.g. `'1.5'` cast to `1`). Change the referencing rows first, or choose a change that keeps the keys equal. |

See [`error-codes.md`](error-codes.md) for the full kind table.

---

## See also

- [Declaring ObjectTypes](declaring-objecttypes.md) — refine the default ObjectType auto-CRUD generates
- [Declaring StateMachines](declaring-state-machines.md) — bind an FSM to an enum column
- [Declaring custom ActionTypes](declaring-actions.md) — write a handler that spans multiple tables
- Phase 17.5 §7 (platform spec) — refinement contract
