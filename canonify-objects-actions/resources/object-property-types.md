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
| `properties.<name>` | A `mapped`, `derived` or `lookup` property refinement (below). |
| `links.<name>` | A typed link — [object-links-and-cardinality.md](object-links-and-cardinality.md). |
| `rollups`, `parts_frozen_when` | Per-reader summaries over to-many links; a compound root's freeze gate — links page. |
| `temporal` | Effective dating, `{ from, to, key }` — below. |
| `display` | How a person sees the type: titles, forms, lists, actions — below. |
| `attachments` | `{ accept?, max_bytes?, kinds? }`: what the record's attachments take, enforced on every attach ([files](../../canonify-isomorphism/resources/runtime-contract.md)); `kinds` names its attachment kinds, each backed by a child type — below. |
| `idPrefix` | Prefix of generated ids. Omitted → `rec`, so ids are `rec_<ULID>`; the generator adds the one `_`, so `ana` and `ana_` both mint `ana_<ULID>`. Changing it later affects new ids only. |
| `defaults` | Per-verb auto-CRUD opt-out: `{ create?, read?, update?, delete?, list?, replace? }`. `false` removes that generated action from every surface (and frees its name for your own `actions.declare`). `replace: false` on a non-compound type is an accepted no-op. |
| `browsable` | `false` hides the type from Browse data and skips its generated `<type>.list` view; access is unchanged. `true` clears it; omitted leaves it alone. |
| `confidential` | Sensitive type — below. |
| `expectedHash` / `force` | Optimistic concurrency on a re-refine — below. |

**Attachment kinds.** A kind is a named set of a record's files that are rows
of a child type: `attachments.kinds.<kind> = { via, kind?, file?, label?,
accept?, max_bytes?, display?, facts? }`. `via` is a to-many link of the type
(the child holds the parent's id); `file` the child's `file` property (default
its only one); `kind` the value of the child's `kind` column its rows of this
kind carry (omit when the child holds this kind alone); `accept` / `max_bytes`
the kind's own file rules; `display: "hero"` an image kind shown as one large
image (needs an image-only `accept`); `facts` child fields shown with each file
(an expiry date). A file of a kind is written through the child's actions — upload
it naming the kind (`POST /v1/files` with `object_type` + `kind`, refused
before a byte is stored), then `<child>.create` with the parent's id, the
kind and the file id — and read by kind (`?kinds=` on the attachments read)
under the child's policy: a reader whose policy withholds the child type or its
file column is told the kind is withheld and shown none of its files. Every
write that stores a file in a kind's row, or changes the row's kind, holds the
file to the kind's rules. Example:

```json
{
  "name": "vehicle",
  "links": {
    "photos": { "target": "vehicle_photo", "cardinality": "many", "via": "vehicle_id" },
    "documents": { "target": "vehicle_document", "cardinality": "many", "via": "vehicle_id" }
  },
  "attachments": {
    "kinds": {
      "photo": { "via": "photos", "accept": ["image/*"], "max_bytes": 5242880, "display": "hero" },
      "certificate": { "via": "documents", "kind": "certificate", "accept": ["application/pdf"], "facts": ["expires_on"] }
    }
  }
}
```

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
| `lookup` | one property of the row a to-one link points at, read per reader | you | no |
| `computed` | a function over the row, never persisted | platform only | no |

A mapped refinement is `{ kind, column, type, nullable?, hidden?, label?,
description?, values?, valueColors?, valueLabels?, currency?, unit?, scale?,
principal?, accept?, max_bytes?, history? }`. Its `column` must exist on the table, else
`Property "x" maps to column "x" which does not exist on table "t"`.

**What changed, and when.** A mapped property declared `history: true` keeps
its changes: every write that changes it (the generated update and replace, a
declared handler's `update` step) records one entry in the record's history —
from, to, the action, who, and when on the org clock — in the write's own
transaction. Every transition is recorded the same way without opting in
([state-machines.md](state-machines.md#the-records-history)). Read it with
`objects.history { object_type, id, limit? }`: newest first, under the
caller's policy — a record they may not `get` is `NOT_FOUND`, and the entries
of a property their columns withhold are left out and the property named in
`withheld` with its entry count. Opt in only for fields whose past matters
(a price, an assignee, a due date): an unchanged value records nothing.

**Who did it.** A `string` property declared `principal: true` holds an actor
id — what `$principal.id` writes. Every read that carries titles adds
`<property>__title`: a member's name in the org (else email), an agent key's
`"<label> (agent)"`, for a caller who may read the org's members; every
surface shows that name, never the id. Any other type is refused. **A
reference (FK)** likewise shows its record's title on every surface — cells,
filter chips, grouped labels, record and card titles, Knowledge scopes —
carried as the FK name minus `_id` plus `__title` (`sample_id` →
`sample__title`); a record titled by a reference reads that record's title. A
principal or FK read through a link carries its name as `<ref>__title` too.

### Derived properties

`{ kind: "derived", type, expression, hidden?, label?, description?, currency?,
unit?, scale?, values?, valueLabels?, valueColors?, principal? }` (a boolean
also `meaning?, on?, amount?, amountLabel?, labelOne?, fix?, fixSubject?` — see Named
conditions below) — no `column`, so no `nullable` either (a derived value is
NULL whenever its expression yields none), and `values` only on a `type:
"enum"` (or `set<enum>`) property. The expression, usually a correlated subquery
through a foreign key, is projected into the read view as
`(<expression>) AS "<property>"` in the same transaction, so the property reads,
filters and renders at once:

```json
{
  "name": "analysis",
  "properties": {
    "client_plan": {
      "kind": "derived",
      "type": "string",
      "label": "Client plan",
      "expression": "SELECT plan FROM client WHERE client.id = analysis.client_id"
    },
    "extra_charges": {
      "kind": "derived",
      "type": "money",
      "currency": "DKK",
      "expression": "SELECT COALESCE(SUM(amount), 0) FROM charge_line WHERE charge_line.analysis_id = analysis.id"
    }
  }
}
```

- A derived refinement replaces only another derived property (`VALIDATION`
  otherwise). A migration adding a column of the same name converts it to
  mapped, typed like the column, keeping only `label` and `description` —
  restate it as mapped, with its enum facets, in the bundle you apply next.
- The SQL is yours, and it is proven when declared: refine, `catalog validate`
  and `catalog apply` prepare the type's read view and refuse an expression
  SQLite cannot compile — `derived_expression_invalid`, naming the property,
  the expression and the SQLite message. It may not read platform tables.
- An expression may reference **another derived property** of the same type —
  `total` is `instrument_amount + analysis.extra_charges` (inside a subquery,
  qualify it with the table and alias a re-read of that table, else
  `derived_own_table_rebound`). It is inlined, so write each fragment once; a
  cycle (`a → b → a`) is `derived_cycle`; a reader withheld it is withheld this.
- **Time reads the org clock: `:today` and `:now`.** `:today` is the org's
  calendar date in its timezone (`2026-10-22`), `:now` its now as a UTC
  instant (`2026-10-23T05:00:00.000Z`); both follow a `demo_now` pin. So
  `day = :today`, `due_at < :now`, `day <= date(:today, '+7 days')`,
  `julianday(:now) - julianday(opened_at)`. SQLite's own clock is real time in
  UTC — wrong in the evening west of Greenwich and blind to a demo pin — so
  `date('now')`, `datetime('now')`, `julianday('now')`, `strftime(…, 'now')`,
  `unixepoch('now')`, `date()` and `CURRENT_DATE` / `CURRENT_TIMESTAMP` are
  refused (`derived_sql_clock`, the message shows the expression rewritten);
  any other parameter is `derived_parameter_unknown`. A list filter, an
  aggregate count and a handler guard on the same derived condition agree.
- **A local date and time is not an instant: `org_instant(date, time)`.** A
  `date` plus a `time` column (a cue at 19:30 on the event day) is wall time
  in the org's zone; compared with `:now` (UTC) as text it is off by the
  zone's offset (an hour in Lisbon in summer, none in winter). Turn it into
  the instant first: `org_instant(event.date, cue.time) < :now` is "the cue's
  time has passed", DST included (each date takes the offset in force that
  day; `NULL` when either is `NULL`). `org_days(from, to)` is a day series as
  a table (`day`, `weekday` with 0 = Sunday, at most 400 rows) for counting a
  span on every day it covers: `(SELECT count(*) FROM org_days(job.start_on,
  job.end_on) d WHERE d.weekday BETWEEN 1 AND 5)`.
- A derived property works wherever a mapped one does on a read path: list
  filters (a bare `?is_overdue=true` param included), aggregate filters,
  `groupBy`, `sum`, lookups, search when `searchable` (or title-named, ≤1000 rows), and a
  handler's `read` / `read_optional` / `read_many` / `read_aggregate` (filter,
  `order_by`, `sum`, `group_by`) and an `insert`'s `as` binding. No handler step
  may write it.
- This is how you reuse the canonical object instead of copying its name into a
  column — see [`canonify-domain-modelling`](../../canonify-domain-modelling/SKILL.md).
  To show one linked value to readers with scoped access, declare a lookup.

### Named conditions

A derived `boolean` is a **named condition**: a business fact (overdue, late,
short, over the limit, mine) defined once on the type and referred to by its
name everywhere — a list filter (`{ "is_overdue": true }`), a view preset, an
aggregate count, a nav badge's `count.filter`, a handler guard
(`{ "op": "assert", "when": { "ref": "$inv.is_overdue", "eq": true } }`).
Every path answers the same rows. A condition holds or it does not: a `NULL`
expression reads `false`, so `{ "is_overdue": false }` is the exact complement.
Declare it on the type instead of repeating the comparison in each view.

It may also say what it means and what resolves it:

```json
{
  "name": "invoice",
  "properties": {
    "is_overdue": {
      "kind": "derived",
      "type": "boolean",
      "expression": "due_on < :today AND paid < amount",
      "label": "overdue invoices",
      "labelOne": "overdue invoice",
      "meaning": "overdue",
      "on": "due_on",
      "amount": "open_amount",
      "amountLabel": "open",
      "fix": "invoice.record_payment"
    }
  }
}
```

| Facet | Meaning |
|---|---|
| `meaning` | `overdue`, `late`, `short`, `over`, `blocked` (critical); `at_risk`, `due_soon`, `full` (warn); `met` (good). Absent: a neutral mark. |
| `on` | The date, timestamp or number field it is about; a date or timestamp gives the distance ("Overdue · 12 d"). A `time` is no instant: put the condition on a derived timestamp `org_instant(<date>, <time>)` ("Late · 12 min" for a cue). |
| `amount`, `amountLabel` | A number or money field saying how much of it a row holds ("Short 2"), and what that counts ("open positions"). |
| `label`, `labelOne` | Also the noun its rows are counted in ("3 overdue invoices"), and the noun for one row ("1 overdue invoice"); the count picks by the org language's plural rules. |
| `fix`, `fixSubject` | The action that resolves it: an action of the type or of its subject, or — with `fixSubject`, a to-one link path such as `"customer"` — of the record that path reaches. Declare the action first. |

Refine refuses (`invalid_condition`, with the fix in the message) an `on` or
`amount` that is not a field of the type of the right type, an `amountLabel`
without an `amount`, a `fixSubject` without a `fix`, and a `fix` that is not a
registered action of the type, its subject or the record `fixSubject` reaches;
an unknown `meaning` is refused at decode with the meanings listed.
`schema describe <type>` lists each condition under `conditions` with its
`level` (critical, warn, good, neutral) and facets. A condition whose
expression reads a column the reader may not read is withheld from that reader
like the column: absent from rows and `conditions`, and refused to their
filters, presets and counts; a guard over it still holds, and says only its own
words.

### Lookup properties

`{ kind: "lookup", link, property, label?, hidden?, description? }` — one
property of the row a to-one link points at (the FK on this type), read under
each reader's grants: `null` where they may not read the linked row, withheld
like a masked column where their read withholds that property, the type or the
FK. Its type and presentation (`values`, labels, colours, `currency`,
`principal`) are the target's. It shows on get, list, search hits and
navigate, filters and sorts a list, and may be a title or template token.

```json
{ "name": "analysis", "properties": { "matrix": { "kind": "lookup", "link": "sample", "property": "matrix" } } }
```

One hop only: a to-many link (use a
[rollup](object-links-and-cardinality.md#rollups)), a lookup of a lookup or an
unknown property is refused at declare. A filter on it works on every read of
a block (search, totals, counts, metric, chart, grouped table); an aggregate's
measure or group-by, a search match, a handler `where` / `order_by` or a row
filter naming it is refused with `lookup_not_compiled`, naming it and the fix:
the link path (`sample.matrix`) or the FK.

## The eleven PropertyTypes

| Type | Stored as | Notes |
|---|---|---|
| `string` | TEXT | |
| `number` | INTEGER/REAL | May carry `currency` (major units, legacy/imported columns; a money-named `REAL` column is the `C10-float-money` audit finding). |
| `money` | INTEGER minor units | MUST declare `currency`. `125000` DKK is 1,250.00. A fraction is refused with `VALIDATION`, never rounded. Accepted wherever `number` is (`sum`/`avg`, lookups, measures, `+ - * /`, rollups); aggregates stay in minor units and name `unit: { "type": "money", "currency": "DKK" }`. Views render it as an amount with no `format` hint; forms and range filters take major units from a person and send minor units. |
| `boolean` | INTEGER 0/1 | See the wire contract. |
| `timestamp` | TEXT, ISO-8601 UTC | Never an epoch integer. Write `Z`, an offset, or no zone: `2026-10-23T09:30` is that time on the org's clock (a DST-overlap time is the earlier instant; a DST-gap time is refused `timestamp_in_dst_gap`). `created_at` / `updated_at` are always typed `timestamp`, so `groupBy: "created_at@month"` works on an unrefined table. |
| `date` | TEXT or `DATE`, `YYYY-MM-DD` | No time, no zone. Range filters, `$now±…` and `:today` tokens resolve to calendar dates; `date_add`/`date_sub` on a date returns a date. |
| `time` | TEXT or `TIME`, `HH:MM` | A time of day (24-hour) on the org's wall clock, no date or zone: working or opening hours. `"8:00"` and `"08:00:00"` are refused. Something that happens is a `timestamp`. |
| `json` | TEXT | See the wire contract. |
| `enum` | TEXT | A string narrowed to `values` (or `type: "string"` with `enum: [...]`, an input's spelling, stored as this). |
| `set<enum>` | TEXT, canonical JSON | A closed subset of `values`, written as an array of distinct members, stored in declaration order (`["vegan","gluten_free"]`), read back as an array. Canonical storage makes the whole set a UNIQUE-key component and an equality operand; filter with `contains` / `has_any` / `has_all`. `values`, `valueLabels`, `valueColors` apply as for `enum`. |
| `file` | TEXT | A `file_…` id from an upload; writing it binds the file to the row ([files](../../canonify-isomorphism/resources/runtime-contract.md)). Optional `accept` (`["image/*", ".pdf"]`) and `max_bytes` refuse another type (`VALIDATION`) or a larger file (`PAYLOAD_TOO_LARGE`, 413) on every write that stores one. One file per property (`multiple` is refused — several files are the record's attachments). Cells show the filename, an image a thumbnail. |

SQL inference on the default shape: `TEXT` → `string`, `INTEGER`/`REAL` → `number`, `DATE` → `date`, `TIME` → `time`, `DATETIME`/`TIMESTAMP` → `timestamp`.

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
passes through; any other stored integer is returned unchanged. A list marks a
true flag ✓ (a card: its label) and a false one not at all; a record reads
Yes/No. Declare a 0/1 flag `boolean`, not `number` ([tables-and-import.md](tables-and-import.md)).

### `hasDefault`

Not declarable: auto-CRUD sets it when the column has a SQL `DEFAULT`. Such a
column is optional on create even if `NOT NULL` (an omitted value lets the
`DEFAULT` apply; an explicit one, `null` included, is written), so the
generated form marks it "(optional)": **do not `DEFAULT` a column the form must
require** (the state column is the exception). `column_modify` cannot drop a
`DEFAULT`; on a live table, declare your own create with the input required
and set `defaults: { create: false }`.

## Enum narrowing

A StateMachine binds only to an `enum` property, so narrowing a state column
is the load-bearing refinement. The validator:

1. accepts only a `string` or `enum` base (`…has base type "number"; cannot narrow to enum`);
2. requires non-empty `values`;
3. when the set is narrower than the live one, checks **every persisted value** is covered (`…cannot be narrowed to enum [open, done] — 3 rows hold a value outside it, e.g. "archived"`) — widen the set or fix the rows first. `catalog plan`, `validate` and `apply` all run it before any write (`rows_violate_refinement`).

Tightening `nullable: false` requires zero NULL rows (`…2 rows hold NULL (e.g. row "crn_1")`); loosening is always allowed.

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

Colour the states a person must act on and leave the rest `neutral`: a
7-state lifecycle gives each open state its own (queued `neutral`, received
`cool`, running `info`, on hold `warm`, complete `good`) and its terminal
states reuse a meaning (released `neutral`, cancelled `hot`). The tokens paint
the row's pill and other enums' dots, board column badges, grouped-table enum dimensions, the queue
urgency rail and chart series split by the enum (when every series colour is
distinct). A `StateMeta.color` takes the same tokens. A key not in `values`, or a token
outside the six (`blue` is not a token), fails with `invalid_enum_color`.

### `valueLabels`

`Record<value, string>` — the text each value renders as (list, detail, chip,
board card, queue fact). Display-only: storage, filters, sorts and presets keep
the raw value (`?where[health_status]=green` still matches "Healthy"). A key
not in `values` or a blank label fails with `invalid_enum_label`. A value with
no label falls back to the humanised token (`green` → `Green`) and the catalog
audit warns; a bound StateMachine's `StateMeta.label` is an equivalent source.

### `valueMeanings` and `ordered`

What each value MEANS, for an enum no state machine governs (a governed
enum's meanings are its states' `meaning` — declaring them here too is
refused). A plain status enum declares state meanings — `live`,
`terminal-good`, `terminal-bad`, `sink` — and `ordered: true` when its
`values` are in lifecycle order:

```json
{
  "name": "appointment",
  "properties": {
    "status": {
      "kind": "mapped",
      "column": "status",
      "type": "enum",
      "values": ["booked", "checked_in", "attended", "no_show", "cancelled"],
      "ordered": true,
      "valueMeanings": {
        "booked": "live",
        "checked_in": "live",
        "attended": "terminal-good",
        "no_show": "terminal-bad",
        "cancelled": "sink"
      }
    }
  }
}
```

A **band** (a score's severe / moderate / mild) declares attention meanings
instead — `critical`, `warn`, `good` — usually with `ordered: true`. One
family per enum. Refused with `invalid_enum_meaning`: a key not in `values`,
an unknown meaning (the refusal lists both families), a mix of families.
Both ride `schema describe` and the schema wire as `valueMeanings` /
`ordered`.

`values`, `valueColors`, `valueLabels`, `valueMeanings`, `ordered`, `currency`,
`unit` and `scale` are inherited when omitted and shed when the property stops
being an enum (or a number).

## `currency`, `unit`, `scale`

`currency` is ISO-4217 (`kr` refused, no default): required on `money`, allowed on `number`,
else `invalid_currency`. A plain `number` may declare `unit` (`"kg"`) and `scale`
(`{ "basis": 10000, "as": "percent" }` reads basis points 550 as "5.5%"; `{ "basis":
1000 }` + `"kg"` reads grams as kg); elsewhere or beside a currency `invalid_number_unit`.
They travel with the data, the locale with the viewer: every cell, fact, card, tile,
measure ("159.5 / 500 kg"), axis and form (typed scaled) writes them; storage, `where`
filters and writes keep stored units; an aggregate (a link one too) names its `unit`.
`scale.decimals` (0–6) sets the digits a number shows (`{ "decimals": 1 }` reads
"4,171.4 kg"; `basis` defaults to 1); an empty scale or fractional decimals is
`invalid_number_unit`. `scale` is always this object, never a bare number
(`"scale": 1` is refused): the digits are `decimals` here and on a SpringML
Aggregate or Stat alike (`decimals={1}`). `unit: "min"` is a duration ("9 h 40 min").
A title token or guard-text placeholder carries unit and decimals ("15.3 h"): never repeat the unit after it (`doubled_unit`).

## Field facts: `better`, `thresholds`, `selfNaming`

A `number` or `money` property (mapped, derived or a lookup's target) may declare
which way is better and the bounds it is toned at:

```json
{
  "name": "job",
  "properties": {
    "load_kg": { "kind": "mapped", "column": "load_kg", "type": "number", "unit": "kg",
                 "thresholds": { "warn": 80, "critical": 100 } },
    "score": { "kind": "mapped", "column": "score", "type": "number",
               "better": "up", "thresholds": { "warn": 70, "critical": 40 } }
  }
}
```

- **`better`** — `"up"` (more is better) or `"down"`. A Stat delta and a `change`
  Value tone by it; sums and Aggregates of the field keep it.
- **`thresholds`** — `{ warn, critical }`, either or both, in stored units (money in
  whole minor units). A value at or above a bound takes its tone; with `better: "up"`,
  at or below. The critical bound lies past the warn bound on the bad side. A Spring
  UI Meter over the field is neutral below the bounds and says "Warning" / "Critical"
  at them; a value the reader may not read is never toned. Bounds tone one row's
  value, never a sum.
- **`selfNaming`** — `true` / `false`: whether a card shows the value bare or with
  its label ("Owned 45"). Default by type: links, text, money, dates and pills name
  themselves; numbers and the type's `display.code` do not.

A direction or bound on a non-number, empty thresholds, bounds in the wrong order or
a fractional money bound is `invalid_field_fact`. Preview outputs (`presentation.preview.outputs`)
take the same `better` and `thresholds`.

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
  Who owns which words on the screen is in
  [bilingual-apps.md](../../canonify-app-authoring/resources/bilingual-apps.md).

## `display`

| Key | Meaning | Omitted → |
|---|---|---|
| `title` | How a row's title is built: `{ property }`, `{ template: "{from_state} → {to_state}" }` (an FK token such as `{sample_id}` reads the linked record's title — the link's own name, `{sample}`, holds no value and reads empty, an enum or state token its value label, a `principal` token the actor's name, a date as a day (`27 Apr 2027`) and a number as stored (never grouped), in the org's locale and zone — in the record header, FK cells, pickers, dialogs and search hits alike; an empty token drops with the text glued to it and one separator, so `{code} · {site} · {owner}` with no site reads `S-12 · Ana`, never `S-12 · —` — the rule every template follows, subtitles and queue rows included), or `{ derived: { via: <fk> } }` (borrow the title of the record the FK points to). Every type must resolve a human title, never a raw id; every FK then renders as its record's title, so never derive a `<fk>_name` shadow. | a name-ish property |
| `title` `fallback` | A `{ property }` or `{ template }` title may name a `fallback` template, read when the title reads empty: `{ "property": "number", "fallback": "Draft · {customer_id}" }`. A field the reader may not read reads empty, so the fallback never depends on a hidden value. A fallback that names no field, a field the type lacks, or repeats the title is `invalid_title_fallback`. | none |
| `code` | The record's code — the eyebrow above its human title ("EV-1187" over "Team lunch"): a string or number property, never the title property (`invalid_display_code`). Withheld from a reader whose columns withhold it. | none |
| `recordView` | The Spring UI `RecordPage` a record of the type opens (rows, links). It must be a RecordPage of this type (`invalid_record_view`); a reader who may not open it gets no link. | the type's first RecordPage by name |
| `subtitle` | Same three strategies; promotes the meaningful key when titles collide. An enum in it (property or template token) shows its value label — picker options, record header. | the collision walk: `subtitle` → `primaryProperties` → other fields → `created_at` → `id` |
| `icon` | A kebab-case Lucide icon name (lucide.dev/icons). The platform pins its Lucide version, so an icon added to Lucide recently is `unknown_icon`; the refusal suggests close names. | none |
| `formFields` | Ordered fields of the type's own create/update forms. | every column minus auto-managed ones, optional fields folded |
| `primaryProperties` | Default list columns and the detail view's lead fields (the rest go under "More details"). | inferred |
| `searchable` | Ordered filter-bar properties. Every search (table, picker, `objects search`, quick find) matches these plus the fields the title and subtitle name, never `id`, and only those the reader may read: each typed word must start a word of one of them ("kobe bog" finds "PT-1001 · Kobe Bogaert"; case and accents ignored). A derived property is searched when listed here, or when the title names it and the type has at most 1000 rows, and a type whose only text is derived still gets a search box. A table or picker search also matches each FK by its linked record's search fields ("Harbour" finds the analysis of that sample), fenced by the caller's read of the target; the palette's cross-type `objects.search` follows only the links a hit's title reads. | string/enum properties among `primaryProperties` |
| `statusProperty` | The enum shown as the row's status pill, a Board's lanes and a state's hue — in Spring UI a Table, List, Item, Board and record header draw it automatically. Declare it only to pick another enum than the default. | the property the type's StateMachine governs, else the first enum named `*status*`/`*stage*`, else none |
| `defaultFilter` | Renderer default for browse and reference-picker lists, e.g. `{ status: active }`. Not policy: typed reads still see every row. | none |
| `entityKind` / `entityKindPlural` | Singular/plural type name in generated copy (headings, "Create …", empty states, sidebar). Declare both: the auto-plural is English-only. Blank → `invalid_entity_kind`. | humanised identifier |
| `actions` | Per-action overrides keyed by full action name: `{ hidden?, label?, emphasis?: primary \| secondary, showWhen? }`. `showWhen` is the [one condition grammar](actions-handler-patterns.md#conditions--one-grammar-for-every-surface) (`[{ field, op, value? }]`) over this type's row, for any action it owns, a generated one included (`analysis.update` only while `status` is `queued`); a `field` that is not a property of the type is refused at declare, `VALIDATION/unknown_show_when_field`. | the action's own `presentation` |

**`display.actions` precedence.** Its `label` and `emphasis` win over the
action's `presentation`; its `showWhen` is ANDed with `presentation.showWhen`
(both must hold to offer the action on a row); `hidden` and `showWhen` remove
the button only — on `<type>.update` they govern the record's inline Edit too,
on `<type>.create` a reference picker's "+ Create" — and the action stays invokable over CLI/REST/MCP under the same governance
(withhold with policy, not these keys). `group`, `destructive`, `confirm` and `success` always come from
the action's `presentation`. Declare the override on the type that owns the
action (`analysis.request` → `analysis`); every surface offering that action reads it
there — record, table, board card, queue row, picker.

## Visibility — `hidden` and `confidential`

- **`hidden`** (mapped or derived) keeps a property out of every default
  selection: inferred table columns and record grid, board cards, "More
  details", the metric strip, the default search (a type with no
  `display.searchable`), a generated form's optional fields and the inline
  editor. A view column, `primaryProperties` or `searchable` that names it
  still uses it. It
  is **not** an authorization boundary: reads still return it and a handler can
  read and write it (withhold data with a policy's `columns`).
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
  the `$as_of` token — a legacy v1 view's `source.filter: { "$as_of": "$now" }`, a
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
| Bad colour token, label, meaning, currency, unit/scale, entity kind, icon | `invalid_enum_color`, `invalid_enum_label`, `invalid_enum_meaning`, `invalid_currency`, `invalid_number_unit`, `invalid_entity_kind`, `unknown_icon` |
| Re-refine without / with a stale hash | `CONFLICT/HASH_REQUIRED`, `CONFLICT/STALE_DECLARATION` |
