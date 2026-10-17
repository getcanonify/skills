# Tables, auto-CRUD and bulk import

A table is one `schema.propose_migration` call. The SQL runs in a transaction
(rolled back on failure); then auto-CRUD registers a default ObjectType and the
generated actions, live on every surface without a restart.

```yaml
action: schema.propose_migration
input:
  description: lead table — pre-conversion sales prospect   # lands in the audit trail
  sql: |
    CREATE TABLE lead (
      id           TEXT PRIMARY KEY,
      name         TEXT NOT NULL,
      email        TEXT NOT NULL,
      status       TEXT NOT NULL DEFAULT 'new',
      customer_id  TEXT REFERENCES customer(id),
      created_at   TEXT NOT NULL,
      updated_at   TEXT NOT NULL
    )
```

Envelope: `{ data: { applied: true, table: "lead" }, audit_event_id }`. One
statement per call; each call commits on its own (a failure on table 2 leaves
table 1 committed). Re-running a `CREATE TABLE` for an existing table is
`VALIDATION`; the action is `per-key` idempotent.

## Column conventions

- **`id TEXT PRIMARY KEY`** holding a prefixed ULID (`lea_…`). A create that
  omits `id` gets one minted with the ObjectType's `idPrefix` (generic `rec`
  without one); a create may pass its own unique `id`. `id` must be unique —
  the PRIMARY KEY or `id TEXT NOT NULL UNIQUE`. A plain `id TEXT`, a composite
  key, a generated `id`, a unique index looser than the column's collation, or
  `CREATE TABLE … AS SELECT` is refused (`id_not_unique`); so is
  `ON CONFLICT REPLACE` on any column (`on_conflict_replace`). A table already in
  either shape blocks every later migration until repaired
  (`details.preexisting` names it), and `catalog.refine_object_type` cannot
  re-point `id` at another column.
- **`created_at` / `updated_at`** are `TEXT NOT NULL` ISO 8601 UTC — never an
  integer epoch. Create stamps both when omitted (explicit values are kept, e.g.
  for an import); every update re-stamps `updated_at`.
- **Money** is an `INTEGER` of minor units (øre, cents), declared `type: money`
  with a `currency` on the ObjectType. Never `REAL`.
- A column a StateMachine will drive is a plain `TEXT NOT NULL DEFAULT
  '<initial>'`; narrow it to an `enum` on the ObjectType.
- **Booleans** are `INTEGER` 0/1 columns refined to `type: boolean` (below).
- `_`-prefixed tables are platform-owned; a table may not take a platform
  ObjectType's name (`reserved_table_name`).

## What auto-CRUD generates

- An ObjectType with one `mapped` property per column, typed from SQL: `TEXT` →
  `string`, `INTEGER` / `REAL` → `number`, `DATE` → `date`, `DATETIME` /
  `TIMESTAMP` and `created_at` / `updated_at` → `timestamp`. Never `enum`,
  `money` or a link — refine those
  ([object-property-types.md](object-property-types.md)).
- `<t>.create`, `<t>.read`, `<t>.update`, `<t>.delete`, `<t>.list`, each
  schema-validating its input and running through the full runtime frame.
  `create` takes the columns as fields (a `NOT NULL` column without a default is
  required — a `DEFAULT` makes it optional, see
  [`hasDefault`](object-property-types.md); `id`, `created_at`, `updated_at`
  optional). `read` / `update` /
  `delete` address the row by `id`.
- **Compound roots** (a `part_of` child link) change three verbs:
  - `<root>.create` takes an optional slot per `part_of` link (an array for
    `many`, an object for `one`) and inserts the whole tree atomically; each
    child's FK is bound to the new root (supplying it yourself is `VALIDATION`).
  - `<root>.replace` (compound roots only) is a whole-compound PUT: children
    with an `id` are updated, without one inserted, and current children absent
    from the list are **deleted by omission** (their other inbound links still
    honour `on_delete`). An absent key for a `part_of` link is refused — send
    `[]` / `null` to delete all. An `id` from another root is `VALIDATION`.
  - A part's own `create` / `update` / `delete` run the root's version anchor
    and its `parts_frozen_when` gate (`CONFLICT`, `details.reason: "FROZEN"`);
    an optional `expected_root_version` makes a stale root a `CONFLICT`.
  - Read a compound back with `canon objects get <type> <id> --expand` (REST
    `?expand=1`); declared `rollups` ride on `get`, `list` and `--expand`.
- Turn a verb off with `defaults` on the refinement (`{ "delete": false }`
  removes `<t>.delete` from every surface; `{ "replace": false }` on a compound
  root; on a non-compound type `replace: false` is a harmless no-op).

**Generated-write semantics.**

- A nullable field takes an explicit `null` on every generated write: create and
  replace store NULL, and `<t>.update` **clears** the field with it (CLI:
  `canon <t>.update --id … --<field> null`). `null` on a NOT NULL field — a NOT
  NULL number included — is refused with `VALIDATION`; it is never stored as `0`.
- A boolean field accepts `true` / `false`, `"true"` / `"false"`, `1` / `0` and
  `"1"` / `"0"`; anything else (`"yes"`, `2`) is refused.
- A UNIQUE, CHECK or NOT NULL violation is `VALIDATION`
  (`details.code: constraint_violation`), and a missing parent id is refused
  naming the field — and the id, when the caller may read the parent type
  unfiltered ([error-codes.md](../../canonify/resources/error-codes.md)).
- A property a StateMachine binds is not writable by create (except its initial
  state), update or replace — see [state-machines.md](state-machines.md).

## Booleans over an INTEGER 0/1 column

Refine a flag-like `number` column (`is_*`, `has_*`, `at_risk`, `active`) to
`type: boolean`: reads return `true` / `false`, the app renders Yes/No, storage
stays 0/1. In a handler, a boolean property takes the literals `0` / `1`,
`null` (if nullable) or a ref that resolves to a **boolean**; a ref to a
`number` (or a numeric expression) is refused at type-check, even when its
values are 0/1. So when you retype a property, retype every action input that
feeds it to `boolean` too — otherwise apply fails with `$input.at_risk resolves
to 'number', but customer.at_risk is declared 'boolean'`.

## Changing a column on a table with rows — `column_modify`

SQLite has no `ALTER COLUMN`. Pass `column_modify` instead of `sql` and the
platform rebuilds the table in one call, keeping every row, the table's own DDL
(only that column changes), indexes, UNIQUE / CHECK / REFERENCES / COLLATE
clauses, triggers, other tables' references and the search index.

```yaml
action: schema.propose_migration
input:
  description: money in minor units
  column_modify: { table: invoice, column: amount, type: INTEGER, cast: round }
```

| Field | Meaning |
|---|---|
| `nullable` | `true` drops NOT NULL, `false` adds it. |
| `type` | The new declared type name (`INTEGER`, `TEXT`, `NUMERIC(10, 2)`). |
| `cast` | Accept a lossy conversion: `round` (nearest integer; INTEGER targets only) or `truncate` (SQLite's CAST, toward zero). |

Without `cast`, a change that would alter a stored value (`31.25` or `'007'`
into INTEGER) is refused with `lossy_conversion` (`rows`, `example_ids`); whole
numbers (`3120.0`) convert freely. Non-numeric text, blobs and out-of-range
numbers are refused even with a cast (`unconvertible_value`). The PRIMARY KEY
cannot change type (`pk_type_change`). Update the bundle's `CREATE TABLE` to
match, or the next apply reports `schema_ddl_divergence`.

## Migration refusals (all `VALIDATION`, nothing ran)

| `details.code` / symptom | Fix |
|---|---|
| bad SQL, existing table, `_` prefix, missing `id`/`created_at`/`updated_at` | fix the SQL |
| `id_not_unique` | `id TEXT PRIMARY KEY`; on an existing table `CREATE UNIQUE INDEX "<t>_id_unique" ON "<t>" (id)` |
| `on_conflict_replace` | drop the clause (a `column_modify` rebuild removes it and keeps rows) |
| `many_statements_in_migration` | one statement per call |
| `system_table_in_migration`, `virtual_table_module_in_migration`, `disallowed_function_in_migration`, `disallowed_statement_in_migration` | migrations create and alter only your own tables: no platform tables, virtual-table modules, non-allow-listed functions, `ATTACH` / `DETACH` / `VACUUM` / `PRAGMA` or transaction control |
| `lossy_conversion`, `unconvertible_value` | add `cast`, or fix the listed rows |
| `foreign_key_violation` (`tables`, `rows`) | a type change broke a referenced key (`'1.5'` cast to `1`); change the referencing rows first |

## Bulk load — `objects.import`

Seed data and history from a system you migrate off load through the built-in
`objects.import`: one governed, audited, atomic call per batch of up to **500
rows** of one ObjectType.

```sh
# a JSON Lines file, a JSON array, or `-` for stdin; the CLI cuts it into
# batches (500, or --batch-size N) and commits them in order
canon objects import job --file jobs.jsonl --on-conflict skip
```

Over REST / MCP it is the action `objects.import` with
`{ object_type, rows: [...], on_conflict }`.

- **Each row is a `<type>.create` input** — same fields, enums, required
  columns, integer money — plus an optional own `id` and `created_at`, and,
  unlike create, **any declared state** of the type's StateMachine, so closed
  work loads closed. An explicit `null` on a nullable field stores NULL; it is
  refused on a required one.
- **All or nothing per batch.** Every row is validated first; one bad row
  refuses the batch with `VALIDATION` (`import_row_invalid`), each issue keyed
  `field_errors["rows.<i>.<field>"]`. A database constraint (a `UNIQUE` column)
  refuses the batch the same way and names the row. The CLI stops at the first
  refused batch, re-keys errors to the row's position in the file and reports
  how many rows earlier batches imported (those stay).
- **`on_conflict`** for an existing `id`: `"error"` (default) refuses the batch;
  `"skip"` leaves the stored row and lists it in `skipped_ids` — with your own
  ids, re-running a failed import is safe.
- **Parents before children.** Import customers, then bikes, then jobs, giving
  parents their ids so children can name them; a part must name its root. A
  missing parent refuses the batch with `VALIDATION`
  (`details.code: constraint_violation`) naming the row (`details.row`) and the
  field (`field_errors["rows.<i>.<field>"]`), and the missing id when the
  caller's read of the parent type is an unfiltered Allow — the full rule is in
  [error-codes.md](../../canonify/resources/error-codes.md).
- **Import is not a create.** No reaction, transition hook or state guard runs
  for an imported row; the call is ONE audit event whose result names
  `imported` and `skipped`.
- **Owner or admin only** (`manage`), with an unrestricted create grant on the
  type; a platform ObjectType is refused.
