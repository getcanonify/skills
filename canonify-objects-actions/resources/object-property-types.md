# ObjectTypes — `catalog.refine_object_type`

Once a table exists, auto-CRUD has registered a permissive default ObjectType
(one `mapped` property per column, typed from its SQL type, no links, no
description — see [tables-and-import.md](tables-and-import.md)).
`catalog.refine_object_type` layers the rich shape on top. This page is the
reference for everything it accepts except links (→
[object-links-and-cardinality.md](object-links-and-cardinality.md)).

**Accumulation.** Organic refinements accumulate: a property, link or display
key omitted from a later call is kept from the earlier one (`display.actions`,
`links` and `rollups` merge key by key). A bundle apply is different: each
`object-types/<name>.json` is the type's whole refinement, so what the file
drops falls back to the inferred shape (the column itself stays).

## Input

| Field | Effect |
|---|---|
| `name` | The ObjectType (required). |
| `description` | Type-level prose shown in `canon objects list-types` and `schema describe`. |
| `properties.<name>` | A `mapped` or `derived` property refinement (below). |
| `links.<name>` | A typed link — [object-links-and-cardinality.md](object-links-and-cardinality.md). |
| `rollups`, `parts_frozen_when` | Compound-root summaries and freeze gate — links page. |
| `temporal` | Effective dating, `{ from, to, key }` — below. |
| `display` | How a person sees the type: titles, forms, lists, actions — below. |
| `idPrefix` | Prefix of generated ids. Omitted → `rec`, so ids are `rec_<ULID>`; the generator adds the one `_`, so `job` and `job_` both mint `job_<ULID>`. Changing it later affects new ids only. |
| `defaults` | Per-verb auto-CRUD opt-out: `{ create?, read?, update?, delete?, list?, replace? }`. `false` removes that generated action from every surface (and frees its name for your own `actions.declare`). `replace: false` on a non-compound type is an accepted no-op. |
| `browsable` | `false` hides the type from Browse data and skips its generated `<type>.list` view; access is unchanged. `true` clears it; omitted leaves it alone. |
| `confidential` | Sensitive type — below. |
| `expectedHash` / `force` | Optimistic concurrency on a re-refine — below. |

Unknown fields fail the decode. One full example:

```json
{
  "name": "task",
  "description": "A unit of work assigned to a member.",
  "idPrefix": "tsk",
  "properties": {
    "status": {
      "kind": "mapped",
      "column": "status",
      "type": "enum",
      "values": ["open", "in_progress", "done", "cancelled"],
      "valueColors": { "open": "info", "in_progress": "warm", "done": "good" },
      "valueLabels": { "in_progress": "In progress" }
    },
    "priority": { "kind": "mapped", "column": "priority", "type": "number", "label": "Priority" }
  },
  "display": { "title": { "property": "title" }, "statusProperty": "status" }
}
```

Success returns `{ data: { name, refined: true, refinement_id }, audit_event_id, … }`.
Verify with `canon schema describe --object task`.

## Property kinds

| `kind` | Backed by | Declared by | Writable |
|---|---|---|---|
| `mapped` | a real column | you | yes |
| `derived` | a scalar SQL expression in the read view | you | no |
| `computed` | a function over the row, never persisted | platform only | no |

A mapped refinement is `{ kind, column, type, nullable?, hidden?, label?,
description?, values?, valueColors?, valueLabels?, currency? }`. Its `column`
must exist on the table, else `Property "x" maps to column "x" which does not
exist on table "t"`.

### Derived properties

`{ kind: "derived", type, expression, label?, description?, currency?, values?,
valueLabels? }` — no `column`. The expression, usually a correlated subquery
through a foreign key, is projected into the read view as
`(<expression>) AS "<property>"` in the same transaction, so the property reads,
filters and renders at once:

```json
{
  "name": "job",
  "properties": {
    "customer_tier": {
      "kind": "derived",
      "type": "string",
      "label": "Customer tier",
      "expression": "SELECT tier FROM customer WHERE customer.id = job.customer_id"
    },
    "parts_cost": {
      "kind": "derived",
      "type": "money",
      "currency": "DKK",
      "expression": "SELECT COALESCE(SUM(amount), 0) FROM part_line WHERE part_line.job_id = job.id"
    }
  }
}
```

- A derived refinement replaces only another derived property; pointing it at a
  mapped or computed name (or refining a derived one as mapped) is `VALIDATION`.
- The SQL is yours: a wrong column surfaces at read time, not declare time. It
  may not read platform tables.
- A derived property works wherever a mapped one does on a read path: list
  filters (a bare `?is_overdue=true` param included), aggregate filters,
  `groupBy`, `sum`, lookups, search when listed in `display.searchable`, and a
  handler's `read` / `read_optional` / `read_many` / `read_aggregate` (filter,
  `order_by`, `sum`, `group_by`) and an `insert`'s `as` binding. No handler step
  may write it.
- This is how you reuse the canonical object instead of copying its name into a
  column — see [`canonify-domain-modelling`](../../canonify-domain-modelling/SKILL.md).

## The ten PropertyTypes

| Type | Stored as | Notes |
|---|---|---|
| `string` | TEXT | |
| `number` | INTEGER/REAL | May carry `currency` (major units, legacy/imported columns; a money-named `REAL` column is the `C10-float-money` audit finding). |
| `money` | INTEGER minor units | MUST declare `currency`. `125000` DKK is 1,250.00. A fraction is refused with `VALIDATION`, never rounded. Accepted wherever `number` is (`sum`/`avg`, lookups, measures, `+ - * /`, rollups); aggregates stay in minor units and name `unit: { "type": "money", "currency": "DKK" }`. Views render it as an amount with no `format` hint; forms and range filters take major units from a person and send minor units. |
| `boolean` | INTEGER 0/1 | See the wire contract. |
| `timestamp` | TEXT, ISO-8601 UTC | Never an epoch integer. `created_at` / `updated_at` are always typed `timestamp`, so `groupBy: "created_at@month"` works on an unrefined table. |
| `date` | TEXT or `DATE`, `YYYY-MM-DD` | No time, no zone. Range filters, `$now±…` and `:today` tokens resolve to calendar dates; `date_add`/`date_sub` on a date returns a date. |
| `json` | TEXT | See the wire contract. |
| `enum` | TEXT | A string narrowed to `values`. |
| `set<enum>` | TEXT, canonical JSON | A closed subset of `values`, written as an array of distinct members, stored in declaration order (`["vegan","gluten_free"]`), read back as an array. Canonical storage makes the whole set a UNIQUE-key component and an equality operand; filter with `contains` / `has_any` / `has_all`. `values`, `valueLabels`, `valueColors` apply as for `enum`. |
| `file` | TEXT | A `_files` id. |

SQL inference on the default shape: `TEXT` → `string`, `INTEGER`/`REAL` →
`number`, `DATE` → `date`, `DATETIME`/`TIMESTAMP` → `timestamp`.

### JSON and boolean wire contract

> For a `json`-typed property, INPUT = a JSON value (array or object) or a JSON
> string that must parse (otherwise `VALIDATION` names the property); STORAGE =
> `TEXT`; OUTPUT on every typed surface (action envelope,
> `objects.get`/`list`/`navigate`/`aggregate`, declared returns, CLI/REST/MCP
> alike) = the parsed value; `sql_read` = raw `TEXT` by contract; booleans
> follow the same rule (`0`/`1` stored, `true`/`false` returned).

Breaking change since `2026.8.28.1409`.

Booleans: writes accept `true`/`false` and the literals `0`/`1` (the
type-check blesses exactly those two literals against a boolean target, e.g.
`set: { at_risk: 1 }`; `2`, `-1` or a `$ref` typed `number` stay a mismatch).
Every read (`get`/`list`/`search`/`navigate`) returns `true`/`false`; `null`
passes through; any other stored integer is returned unchanged. The UI renders
Yes/No. Declare a 0/1 flag column `boolean`, not `number` — retyping one is in
[tables-and-import.md](tables-and-import.md).

### `hasDefault`

Not declarable: auto-CRUD sets it when the column has a SQL `DEFAULT`. Such a
column is optional on create even if `NOT NULL`, and an omitted value lets the
`DEFAULT` apply (the create echo re-reads the stored row). An explicit value,
including `null`, is always written. The generated create form therefore marks
it "(optional)", and no refinement makes it required: **do not `DEFAULT` a
column the form must require** (the state column is the exception).
`column_modify` cannot drop a `DEFAULT`; on a live table, declare your own
create with the input required and set `defaults: { create: false }`.

## Enum narrowing

A StateMachine binds only to an `enum` property, so narrowing a state column
is the load-bearing refinement. The validator:

1. accepts only a `string` or `enum` base (`…has base type "number"; cannot narrow to enum`);
2. requires non-empty `values`;
3. checks **every persisted value** is covered (`…cannot be narrowed to enum [open, done] — existing row has value "archived"`) — widen the set or fix the rows first.

Tightening `nullable: false` requires zero NULL rows (`…N row(s) currently
hold NULL`); loosening is always allowed.

### `valueColors` — the token table

`Record<value, token>`; every key must be in `values`. The tokens are the one
closed, white-label palette (the brand accent flows through them):

| Token | Role | Use for |
|---|---|---|
| `hot` | danger | urgent, destructive, failed |
| `warm` | warning | needs attention, waiting |
| `good` | success | done, healthy, ready |
| `cool` | calm | a calm steel blue — booked, scheduled, low; the cool end of `hot` → `warm` → `cool`, never success (that is `good`) |
| `info` | informational | in progress |
| `neutral` | muted | default; also the colour of a value with no entry |

Colour the states a person must act on and leave the rest `neutral`. The six
are distinct colours in light and dark, so a 7-state lifecycle gives each open
state its own (booked `neutral`, checked in `cool`, in progress `info`, waiting
`warm`, ready `good`) and its terminal states reuse a meaning (collected
`neutral`, cancelled `hot`). The tokens paint status chips, board column
badges, grouped-table enum dimensions, the queue card's urgency rail, and
chart series split by the enum (when every series colour is distinct; otherwise
the chart palette). A `StateMeta.color` takes the same tokens. A key not in
`values`, or a token outside the six (`blue` is not a token), fails
`catalog.apply` with `invalid_enum_color`.

### `valueLabels`

`Record<value, string>` — the text each value renders as (list, detail, chip,
board card, queue fact). Display-only: storage, filters, sorts and presets keep
the raw value (`?where[health_status]=green` still matches "Healthy"). A key
not in `values` or a blank label fails with `invalid_enum_label`. A value with
no label falls back to the humanised token (`green` → `Green`) and the catalog
audit warns; a bound StateMachine's `StateMeta.label` is an equivalent source.

`values`, `valueColors`, `valueLabels` and `currency` are inherited when a
re-refine omits them and shed when the property stops being an enum (or an
amount).

## `currency`

An ISO-4217 code (`/^[A-Z]{3}$/`; `kr` is refused). Required on `money`
(mapped or derived), allowed on `number`, refused on any other type — all
`invalid_currency`. There is no default currency: the currency travels with the
data, the locale with the viewer. It differs from a ViewSpec `format:
"currency"` hint, which only renders. Aggregates keep it (`unit`), and a link
aggregate (`jobs.sum:revenue`) renders in the target property's currency.

## Labels, descriptions, localization

- **`label`** — the field's human name in every header, form field and filter.
  Any type, mapped or derived. Display-only (the property name stays the API
  contract), inherited when omitted, survives a type change. Blank →
  `invalid_property_label`. Without it the name is humanised (`default_price` →
  "Default Price").
- **`description`** — prose about the field, surfaced in `schema describe` and
  discovery as `fieldDescriptions`; never used as a header.
- `label`, `valueLabels`, `entityKind`/`entityKindPlural` and `display.actions`
  labels may be literals or `@t:<key>` message keys
  ([bilingual-apps.md](../../canonify-app-authoring/resources/bilingual-apps.md)).
  Who owns which words on the screen, and the fallbacks, is one table in
  [`canonify-viewspecs`](../../canonify-viewspecs/SKILL.md).

## `display`

| Key | Meaning | Omitted → |
|---|---|---|
| `title` | How a row's title is built: `{ property }`, `{ template: "{from_state} → {to_state}" }` (an FK token such as `{bike_id}` reads the linked record's title — in the record header, FK cells, pickers and search hits alike), or `{ derived: { via: <fk> } }` (borrow the title of the record the FK points to). Every type must resolve a human title, never a raw id; every FK then renders as its record's title, so never derive a `<fk>_name` shadow. | a name-ish property |
| `subtitle` | Same three strategies; promotes the meaningful key when titles collide. | the collision walk: `subtitle` → `primaryProperties` → other fields → `created_at` → `id` |
| `icon` | Lucide icon name (`unknown_icon` otherwise). | none |
| `formFields` | Ordered fields of the type's own create/update forms. | every column minus auto-managed ones, optional fields folded |
| `primaryProperties` | Default list columns and the detail view's lead fields (the rest go under "More details"). | inferred |
| `searchable` | Ordered filter-bar properties. Search matches every mapped property but `id`; a derived property is searched (substring) only when listed here, and a type whose only text is derived still gets a search box. A table search also matches each FK by its linked record's searchable text ("Trek" finds the job on that bike), fenced by the caller's read of the target; the palette's cross-type `objects.search` follows only the links a hit's title reads. | string/enum properties among `primaryProperties` |
| `statusProperty` | The enum shown as the status pill. | first enum named `*status*`/`*stage*`, else none |
| `defaultFilter` | Renderer default for browse and reference-picker lists, e.g. `{ status: active }`. Not policy: typed reads still see every row. | none |
| `entityKind` / `entityKindPlural` | Singular/plural type name in generated copy (headings, "Create …", empty states, sidebar). Declare both: the auto-plural is English-only. Blank → `invalid_entity_kind`. | humanised identifier |
| `actions` | Per-action overrides keyed by full action name: `{ hidden?, label?, emphasis?: primary \| secondary, showWhen? }`. `showWhen` is the forms predicate grammar (`[{ field, op, value? }]`) over this type's row, for any action it owns, a generated one included (`job.update` only while `status` is `open`); a `field` that is not a property of the type is refused at declare, `VALIDATION/unknown_show_when_field`. | the action's own `presentation` |

**`display.actions` precedence.** Its `label` and `emphasis` win over the
action's `presentation`; its `showWhen` is ANDed with `presentation.showWhen`
(both must hold to offer the action on a row); `hidden` and `showWhen` remove
the button only — on `<type>.update` they govern the record's inline Edit too —
and the action stays invokable over CLI/REST/MCP under the same governance
(withhold with policy, not these keys). `group`, `destructive`, `confirm` and `success` always come from
the action's `presentation`. Declare the override on the type that owns the
action (`job.book` → `job`); every surface offering that action reads it
there — record, table, board card, queue row.

## Visibility — `hidden` and `confidential`

- **`hidden`** (per mapped property) keeps a column out of the typed read shape
  and default pages; `hidden: false` surfaces a column the platform hid (e.g.
  `org_id`) so a handler can write through it. It is **not** an authorization
  boundary: a handler that names it can still read and write it.
- **`confidential: true`** (per type) marks sensitive data. Under production
  deny-by-default an ungranted access is still `FORBIDDEN` with audit
  `status='denied'`; a **granted mutation** is audited `status='sensitive'`
  (`canon audit --status sensitive`). Granted reads persist no audit row
  (reads are audited on denial only), so there is no trail of who read it.

```json
{
  "name": "customer",
  "confidential": true,
  "properties": {
    "org_id": { "kind": "mapped", "column": "org_id", "type": "string", "hidden": false },
    "internal_notes": { "kind": "mapped", "column": "internal_notes", "type": "string", "hidden": true }
  }
}
```

## Effective-dated types — `temporal`

For anything in force from one date until another (prices, standing orders,
ratios):

```json
{
  "name": "price",
  "temporal": { "from": "valid_from", "to": "valid_to", "key": ["product_id", "customer_id"] }
}
```

- Rows are half-open periods `[from, to)` per `key`. `from`/`to` are
  `string`/`timestamp` mapped properties, `to` nullable (`NULL` = still open);
  `key` is a non-empty list of mapped properties (never `id`, never a bound). A
  `NULL` key member is a real value (a list price has no `customer_id`).
- **One open period per key**, enforced by a partial `UNIQUE` index
  (`WHERE to IS NULL`) the declaration creates; a table already holding two open
  periods for a key refuses the declaration (`temporal_open_rows_conflict`).
- **`<type>.set`** closes the key's open period at the new `from` (abutting),
  inserts the row, returns it plus `closed_id`. An intersecting period, open or
  closed, is `VALIDATION` / `details: { code: 'temporal_overlap',
  conflicting_id }`; `to <= from` is `temporal_empty_period`. A period the caller
  may not read is never named (no `conflicting_id`, no `closed_id`).
- **`<type>.end`** takes `to` plus `id` or the key: `temporal_end_before_start`,
  `temporal_not_open` (the `id` is closed), `NOT_FOUND` /
  `temporal_no_open_period`.
- **Reads** take `as_of` (`canon objects get|list|navigate … --as-of
  2026-04-01`, REST `?as_of=`, MCP `as_of`; a date, ISO date-time or `$now`);
  `get` reads `null` when the row is not in effect. **Every filter map** takes
  the `$as_of` token — a ViewSpec `source.filter: { "$as_of": "$now" }`, a
  handler `where`, a REST `?where={"$as_of":"2026-04-01"}` — compiling to
  `from <= D AND (to IS NULL OR to > D)`. On a non-temporal type both are
  `as_of_not_temporal`.
- Both verbs are generated like auto-CRUD (every surface, a conformance fixture,
  never exported into `actions/`). A hand-declared action with the same name
  (`price.set` with extra guards) replaces it; the other stays generated. A bad
  spec is `temporal_invalid`.

## Re-refining — `expectedHash` / `force`

The first refinement of a type is free. A re-refine must carry
`expectedHash` (the `declaration_hash` from `catalog.export`'s ObjectType
record; `schema describe` does not show it) or `force: true`:

| Situation | Result |
|---|---|
| neither | `CONFLICT/HASH_REQUIRED` with `details.current_hash`, before any write — even for identical content |
| hash is stale | `CONFLICT/STALE_DECLARATION`, naming the drifted top-level keys (`['properties','links']`) |
| both | `VALIDATION/CONTRADICTORY_DECLARATION_INTENT` |

The same `idempotency_key` resubmitted replays the cached envelope.

## Failure modes

| Symptom | Code |
|---|---|
| Type not registered (declare the table first) | `NOT_FOUND` |
| Enum misses a persisted value / NULL rows on a tightening / unknown column | `VALIDATION` (message names it) |
| Link target or `via` column missing | `VALIDATION` |
| Bad colour token, label, currency, entity kind, icon | `invalid_enum_color`, `invalid_enum_label`, `invalid_currency`, `invalid_entity_kind`, `unknown_icon` |
| Re-refine without / with a stale hash | `CONFLICT/HASH_REQUIRED`, `CONFLICT/STALE_DECLARATION` |
