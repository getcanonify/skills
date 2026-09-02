# Object property types

An ObjectType is the typed shape over one table. Each entry in its
`properties` map is a **property** with a `kind` and a `type`. This page is
the contract for what `catalog.refine_object_type` accepts — verified against
`packages/canon/src/types.ts` (the property interfaces) and
`packages/canon/src/actions/catalog/refine-object-type.ts` (the input schema
+ the `validateRefinement` rules).

## Property kinds

Three kinds exist (`types.ts` — `MappedProperty` / `DerivedProperty` /
`ComputedProperty`):

| kind | backed by | who declares it | writable in a handler |
|---|---|---|---|
| `mapped` | a real column on the table | you, via `catalog.refine_object_type` | yes |
| `derived` | a SQL expression projected into the read view | you, via `catalog.refine_object_type` | no (read-only) |
| `computed` | a synchronous function over the row | platform-tier registration | no (never persisted) |

`catalog.refine_object_type` accepts **`mapped` and `derived`** refinements
(`ObjectPropertyRefinement` is a union of the two). `computed` is a
platform-tier shape you can read in `canon schema describe --object <name>` but
cannot declare.

### Declaring a `derived` property

A derived refinement is three fields — no `column`, since there isn't one:

```json
{
  "name": "customer",
  "properties": {
    "organization_name": {
      "kind": "derived",
      "type": "string",
      "expression": "SELECT name FROM organization WHERE organization.id = customer.organization_id"
    }
  }
}
```

`expression` is a **scalar SQL expression** — usually a correlated subquery
through a FK column. The platform projects it into the ObjectType's read view as
`(<expression>) AS "<property>"` and refreshes the view in the same transaction,
so the property is queryable, filterable and renderable immediately after the
refinement returns. It is read-only: no handler step may write it.

Two rules:

- A derived refinement can only **replace another derived property**. Pointing
  one at an existing `mapped`/`computed` name fails with `VALIDATION`
  (`"…is <kind>; a derived refinement can only replace another derived
  property"`), and refining a `derived` property as `mapped` fails the other way
  (`"…is derived; refinement only supports mapped properties"`).
- The expression is yours to get right — it is SQL against the org's schema, so
  a wrong column name surfaces at read time, not at declare time.

This is the mechanism behind **Obligation 1** — reuse the canonical object.
Instead of retyping a customer's name into a `TEXT` column on every record that
mentions one, declare a link and read the name *through* it. See
[`canonify-domain-modelling`](../../canonify-domain-modelling/SKILL.md).

## The seven PropertyTypes

`PropertyType` (`types.ts`) is exactly:

`string` · `number` · `boolean` · `timestamp` · `json` · `enum` · `file`

- `timestamp` — an ISO-8601 UTC string column (never an epoch integer).
- `json` — a TEXT column holding serialized JSON.
- `enum` — a string column narrowed to a fixed `values` set (see below).
- `file` — a string column holding a `_files` id; treated as TEXT everywhere
  at the typed layer.

## A mapped property refinement (full input + envelope)

This is the input you hand `catalog.refine_object_type` to narrow a status
column to an enum and attach a description. The validator checks the proposed
`values` set against every distinct value already in the column.

```json
{
  "name": "task",
  "description": "A unit of work assigned to a member.",
  "properties": {
    "status": {
      "kind": "mapped",
      "column": "status",
      "type": "enum",
      "values": ["open", "in_progress", "done", "cancelled"]
    },
    "priority": {
      "kind": "mapped",
      "column": "priority",
      "type": "number"
    }
  }
}
```

Expected envelope (success):

```json
{
  "data": { "name": "task", "refined": true, "refinement_id": "re_…" },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

The `refinement_id` and `audit_event_id` are generated; everything else is
deterministic. `catalog.refine_object_type` is `idempotency: per-key` at the
transport layer (a resubmit with the SAME `idempotency_key` replays the
cached envelope, free). Resubmitting the narrowing WITHOUT an
`idempotency_key` is a different story: a re-refine of an already-refined
type is guarded by optimistic concurrency (`expectedHash`/`force`,
default-on) — a bare resubmit with neither is refused before any write
(`CONFLICT/HASH_REQUIRED`), even when the content is identical. Pass the
hash read from `catalog.export`'s refinement record as `expectedHash` (or
`force: true`) to actually land the no-op UPSERT. See
[declaring-objecttypes.md](../../canonify/resources/declaring-objecttypes.md#optimistic-concurrency--expectedhash--force-on-re-refine)
for the full contract.

## Enum narrowing rules

From `validateRefinement` — narrowing `string → enum` is gated:

1. **Base must be `string` or `enum`.** You cannot narrow a `number`/
   `timestamp` column to an enum: `"…has base type "number"; cannot narrow to
   enum (only string and enum bases are supported)"`.
2. **`values` must be non-empty.** Omitting it (or `[]`) →
   `"…refined to enum but no values were supplied"`.
3. **Every persisted value must be covered.** If a row already holds a value
   outside the set, the narrow is refused with the offending value cited:
   `"…cannot be narrowed to enum [open, done] — existing row has value
   "archived""`. Widen the set or fix the data first.

Enum narrowing is the prerequisite for binding a Phase 18.5 StateMachine — a
StateMachine can only bind to an `enum`-typed property.

## Display labels and localization

The property declaration owns tenant-facing vocabulary. For
**generated-from-catalog** form fields and columns, set the property's
display-only `label`; use a literal for fixed copy or `@t:<key>` for a value in
the org's locale bundle. When it is absent, the platform's acronym-aware
English humanizer is retained as a compatibility fallback and a localized
Canon should treat that as migration debt.

For **enum values**, set `valueLabels` next to `valueColors`:

```json
{
  "name": "customer",
  "properties": {
    "lifecycle": {
      "kind": "mapped",
      "column": "lifecycle",
      "type": "enum",
      "values": ["onboarding", "churned"],
      "valueLabels": {
        "onboarding": "@t:customer.lifecycle.onboarding",
        "churned": "@t:customer.lifecycle.churned"
      },
      "valueColors": {
        "onboarding": "blue",
        "churned": "red"
      }
    }
  }
}
```

An omitted enum entry falls back to the English humanized raw value and a
neutral color; the platform locale catalog does not translate tenant enum
values. **Fixed platform chrome** and **action labels** have different owners:
chrome comes from the platform locale catalog, while action captions come from
ActionType/ObjectType/ViewSpec declarations. See the shared string ownership
contract for
the fallback rules and migration note.

## Nullable tightening

Toggling `nullable: false` on a previously-nullable property requires that no
row currently holds NULL in that column. Otherwise:
`"…cannot be tightened to non-nullable — N row(s) currently hold NULL"`.
Going the other way (loosening to nullable) is always allowed.

## `boolean` over an INTEGER 0/1 column

SQLite has no native boolean type — it stores true/false as the integers `1`/`0`.
So a `boolean` property maps cleanly over a plain **`INTEGER`** column (typically
`INTEGER NOT NULL DEFAULT 0`), and the platform bridges the two encodings for
you end-to-end:

- **Writes accept booleans _and_ the literals `0`/`1`.** A handler may `set` the
  property to a JS `true`/`false`, or to the numeric literal `0` / `1` — the
  handler type-check **blesses those two literals** against a boolean target (the
  `flag_at_risk` / `clear_at_risk` shape, `set: { at_risk: 1 }`), and an INTEGER
  `DEFAULT 0`/`1` synthesised into a write flows through the same way. The bless
  is deliberately narrow: **only `0` and `1`**. Any other number literal (`2`,
  `-1`), or a `$ref` that merely resolves to `number`, stays a type mismatch — the
  gate keeps rejecting genuinely wrong data.
- **Reads rehydrate `0`/`1` → `false`/`true`.** Every read surface —
  `get` / `list` / `search` / `navigate` — converts the stored integer back to a
  real JS boolean for a boolean-typed property, so REST JSON and the
  discovery-driven web receive `true`/`false`, not `0`/`1`. `null` (a nullable
  boolean) passes through untouched; **any non-0/1 value is left unchanged** —
  the platform blesses only the canonical encoding and never coerces a stray
  integer by truthiness.
- **The UI renders `Yes` / `No`** for a boolean value instead of a bare `0`/`1`.

Declaring `type: 'boolean'` (rather than leaving a 0/1 column as `number`) is
therefore the right call for a flag column — it's what earns Yes/No rendering and
real booleans on the wire. A `number`-typed property whose *name* reads like a
flag (`is_*`, `has_*`, `at_risk`, …) draws the `elegance_numeric_boolean`
advisory nudging you to switch it. See the `canonify` skill's
[error-codes › *`elegance_*`*](../../canonify/resources/error-codes.md).

## Adding a property the default shape hid

Auto-CRUD's default ObjectType maps one `mapped` property per column, but the
platform may hide physical columns (e.g. `org_id`) from the typed shape. To
let a declared handler write through such a column, **add it as a mapped
property** — the validator permits a new property as long as its `column`
actually exists on the table:

```json
{
  "name": "customer",
  "properties": {
    "org_id": { "kind": "mapped", "column": "org_id", "type": "string", "hidden": true }
  }
}
```

If the column does not exist:
`"Property "org_id" maps to column "org_id" which does not exist on table
"customer""`. (Visibility — the `hidden` flag — is covered in
[object-visibility.md](object-visibility.md).)

## `hasDefault` — columns with a SQL DEFAULT (read-only metadata)

A `MappedProperty` (`types.ts`) can carry `hasDefault?: boolean`. **You don't
declare it** — `ObjectPropertyRefinement` has no `hasDefault` field. Auto-CRUD's
`generateDefaultObjectType` populates it by introspection: when a column's
`PRAGMA table_info.dflt_value` shows a SQL `DEFAULT` clause, the generated
property gets `hasDefault: true`. (Hand-declared ObjectTypes omit it and keep the
prior behaviour.)

It changes one thing — how auto-CRUD `create` treats the column. A `hasDefault`
column is **optional on create** (NOT NULL or not), and when the caller omits it
the generated `INSERT` **drops the column entirely** so the SQL `DEFAULT` applies,
rather than binding an explicit `NULL` that would clobber the default. A supplied
value (including an explicit `null`) is always honoured. So a `status TEXT NOT
NULL DEFAULT 'open'` column does not force every `create` caller to pass
`status`; omitting it stores `'open'`, and the create echo reflects the stored
row (the handler re-reads it so the envelope carries the DEFAULT-supplied value).

## `created_at` / `updated_at` are auto-typed `timestamp`

Auto-CRUD's default ObjectType generator special-cases these two column
names: regardless of how the underlying SQL column is declared, `created_at`
and `updated_at` come out `type: "timestamp"` in the generated ObjectType,
never falling through to the generic SQL-type mapping (which would otherwise
type a `TEXT` column as `string`). The executor stamps both with an ISO-8601
string on every write no matter how the property is typed, so this just makes
the declared type match what is actually written.

The practical effect: a fresh, un-refined table can `groupBy:
"created_at@month"` (or `updated_at@month`) in a `view_specs.declare`
straight away — no `catalog.refine_object_type` step to retype the column to
`timestamp` first. If a `view_specs.declare` groupBy still names a
non-`timestamp` property, check whether the column was hand-declared with a
different name than `created_at`/`updated_at`, or whether a refinement
narrowed it away from `timestamp`; the declare-time error names the
column's actual type either way.

## See also

- [object-links-and-cardinality.md](object-links-and-cardinality.md) — typed links + `on_delete`.
- [object-visibility.md](object-visibility.md) — `hidden` + `confidential`.
- Base skill: `canon skill --name canonify resource declaring-objecttypes.md`.
