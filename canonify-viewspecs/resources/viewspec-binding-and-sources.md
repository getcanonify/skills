# Binding and sources

A block is inert until it's bound to the catalog. This is the grammar for
*what* a block reads: sources, columns, filters, and the render context.

---

## Two source shapes

Both name an ObjectType resolved against the live registry at render time. A
ViewSpec never names a SQL table or a query — only declared catalog pieces.

### ObjectTypeSource — "show these records"

```json
{
  "type": "table",
  "source": {
    "objectType": "customer",
    "filter": { "tier": "gold", "created_at": { "gte": "$now-30d" } },
    "sort": "name"
  },
  "columns": ["name", "email", "tier"]
}
```

- `objectType` (required) — the ObjectType name.
- `filter` (optional) — typed equality scalars, range / set / not-equal
  predicates, or the `$principal` token (below). Keys use the closed path
  grammar `property`, `link.property`, or `link.link.property`; every
  non-terminal link is to-one and a to-many path fails with
  `link_path_to_many`. Literal types are checked against the terminal property
  at `view_specs.declare` time.
- `sort` (optional) — a property name to sort by. Prefix with `-` for
  descending: `"name"` → ascending, `"-name"` → descending. The key must
  be a real property on the ObjectType (validated at declare time).
- `page_size` (optional, and it lives on the `table`/`queue` **block itself**,
  not on `source`) — integer 1–200, default 50; sizes the first page and each
  "Load more" page.

### AggregateSource — "reduce these records to a figure"

```json
{
  "type": "table",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "groupBy": "stage",
    "filter": { "active": true }
  },
  "columns": ["stage", "amount"]
}
```

- `aggregate.op` — one of `count`, `sum`, `avg`, `min`, `max`.
- `aggregate.property` — **required** for sum/avg/min/max; **forbidden** for
  count. `sum`/`avg` additionally require a numeric property.
- `groupBy` (optional) — one property → one row per group (a grouped table).
  Leave it off for a single scalar (used by `metric`). Over a `timestamp`
  property it may carry a time-bucket tail — `"created_at@month"` — to split
  by day/week/month/quarter/year instead of by raw instant. Full grammar +
  rules in [viewspec-analytics](viewspec-analytics.md).
- `seriesBy` (optional) — a **second, categorical** group dimension, splitting
  `groupBy` into series. On a `chart` it draws multiple bars/lines with a
  legend; the same source used by a grouped `table` grows a series column
  instead. Requires `groupBy` and a different property. See
  [viewspec-analytics § seriesBy](viewspec-analytics.md).
- `filter` (optional) — the same equality, range, set-membership, and
  not-equal grammar as `ObjectTypeSource.filter`; literals are checked against
  the terminal property's declared type. Path filters use the same closed
  grammar as columns: `property`, `link.property`, or `link.link.property`;
  every non-terminal segment must resolve to a declared to-one link. A
  to-many link is never legal in a filter key; declaration fails with
  `link_path_to_many`. Unknown segments fail with `unknown_column_ref`.

Path filters and `groupBy` share one resolver and the same two-hop ceiling:

```json
{
  "type": "table",
  "source": {
    "objectType": "order_line",
    "filter": {
      "order.location_id": "L1",
      "order.delivery_date": "2026-09-07",
      "order.status": "locked"
    },
    "aggregate": { "op": "count" },
    "groupBy": "product.name"
  },
  "columns": ["product.name"]
}
```

The terminal is typed from the ObjectType reached by the path. That means an
ISO date string is checked against `order.delivery_date`, not against the
root `order_line` type, and an enum literal is checked against the terminal
enum. A path ending in a link, a path with more than two link hops, or a path
that traverses a to-many link is rejected at declaration — it is never
silently converted into a client-side lookup.

Every traversed ObjectType is policy-gated exactly like a direct read of it.
The join target's row filter is applied in the join and its column projection
gates every referenced column; a principal who cannot read the target gets a
loud read failure rather than a widened or partially-authorized result. This
also applies to the ObjectType named by a measure lookup.

The renderer dispatches a `table` on the presence of `aggregate`: present →
grouped summary table; absent → record list.

```
ObjectTypeSource                AggregateSource (groupBy)
+----------------+              +-------------------+
| one row per    |              | one row per GROUP |
| record         |              | + a measure       |
+----------------+              +-------------------+
| Acme | ops@..  |              | won   | € 1.2M    |
| Beta | hi@..   |              | lost  | € 340k    |
+----------------+              +-------------------+
```

---

## Column references

Columns use a compact dot-traversal grammar. Identifiers are
lowercase-snake. The column's own `ref` is checked at declare time against the
source ObjectType (or the linked target for a valid traversal), and predicate
fields in `emphasizeWhen` / `emphasize[].when` must be direct properties on the
same source ObjectType.

| Form | Meaning |
|---|---|
| `"email"` | A property on the source ObjectType |
| `"customer.tier"` | A property reached through a link named `customer` |
| `"appointments.count"` | Count of rows on the link `appointments` |
| `"orders.sum:amount"` | Sum of the numeric `amount` on the link `orders` |
| `{ "ref": "name", "label": "Full name", "sortable": true }` | A property with a label / sort hint |
| `{ "ref": "amount", "format": "currency" }` | Format hint — `"currency"`, `"percent"`, or `"date"` |
| `{ "ref": "amount", "emphasizeWhen": [{ "field": "status", "op": "eq", "value": "overdue" }] }` | Conditional row emphasis (same Predicate grammar as `showWhen`) |
| `{ "ref": "status", "emphasize": [{ "when": [...], "level": "danger" }] }` | Semantic status emphasis — ordered `good`/`warn`/`danger` rules, first-match wins |
| `{ "ref": "amount", "present": "stat" }` | Alternate presentation — `"stat"` (large figure), `"progress"` (enum step meter), or `{ "kind": "progress", "max": N }` (bounded-number meter) |
| `{ "ref": "amount", "width": "120px" }` | Column width hint — a token (`xs`\|`sm`\|`md`\|`lg`\|`xl`) or a CSS length/fraction (`120px`, `2fr`, `10rem`, `25%`, `8ch`); a bad value fails decode |

Link aggregates: `count` takes no `:property`; `sum` / `avg` / `min` / `max`
each require a `:property` that exists on the link target (and is numeric for
sum/avg). All of this is checked at declare time against the live registry —
a bad ref fails with `unknown_column_ref`.

**`sortable: true` only works on a bare property ref.** A relation-path column
ref (one containing a dot, like `"customer.tier"` or `"orders.sum:amount"`) is a
display projection through a linked ObjectType: it renders, but `runtime.list`
sorts source-ObjectType properties only and does not perform relation joins or
per-row reads. Relation-path sorting is therefore not supported by design.
Declaring `sortable: true` on one is rejected at declare time with
`unsupported_sortable_column_ref` — the web would otherwise offer a sort control
whose first click always fails at read. If the related value must be sortable,
denormalize it onto the source ObjectType; otherwise sort by a local property.
Apply `sortable: true` to own-type properties only (like
`{ "ref": "name", "sortable": true }`).
**Migration note:** bundles authored before 2026-08-06 may include relation-path
`sortable: true` columns; edit the bundle to remove the `sortable` hint on those
columns. This is an intentional platform rule, not a temporary gap.

```json
{
  "type": "table",
  "source": { "objectType": "customer" },
  "columns": [
    "name",
    { "ref": "tier", "label": "Loyalty", "sortable": true },
    "appointments.count",
    "orders.sum:amount"
  ]
}
```

```
+-----------+----------+------------+----------------+
| Name      | Loyalty  | Appts (#)  | Orders (Σ amt) |
+-----------+----------+------------+----------------+
| Acme Corp | Gold     |    12      |   € 48,200     |
+-----------+----------+------------+----------------+
```

### Format hints and conditional emphasis (R7a)

`format` renders a numeric or timestamp column in a specific style. `"currency"` and
`"percent"` require a numeric property; `"date"` requires a timestamp.

`format: "date"` truncates to the **calendar date only** — never a
time-of-day — even over a `timestamp` property, which always stores a full
ISO-8601 datetime. The truncation is UTC-pinned, so the displayed day matches
the stored date for every viewer regardless of local timezone (a bare
UTC-midnight value never slips back a day for a viewer west of UTC). A column
that should show the time too needs no `format` hint — the default timestamp
rendering already includes it.

`format: "currency"` is a **viewer-locale rendering** hint — it decides how to
*show* a number (symbol placement, grouping), **not** what the amount is
denominated in. The currency an amount is *denominated in* is a fact about the
data, declared on the ObjectType property itself via `currency` (see the base
`canonify` skill's *Property currency*). Currency travels with the data; locale
with the viewer.

```json
{
  "type": "table",
  "source": { "objectType": "invoice" },
  "columns": [
    "name",
    { "ref": "amount", "format": "currency" },
    { "ref": "tax_rate", "format": "percent" },
    { "ref": "due_date", "format": "date" }
  ]
}
```

`emphasizeWhen` highlights a cell when ALL listed predicates hold for that row.
It reuses the **same Predicate grammar** `showWhen` uses — no new vocabulary.
One treatment only: a single highlight style. Every predicate `field` is
validated at declare time as a real property on the source ObjectType; a typo
is rejected with `unknown_column_ref` instead of becoming a permanently
inactive highlight.

```json
{
  "type": "table",
  "source": { "objectType": "invoice" },
  "columns": [
    "name",
    {
      "ref": "amount",
      "format": "currency",
      "emphasizeWhen": [{ "field": "status", "op": "eq", "value": "overdue" }]
    },
    "status"
  ]
}
```

### Semantic emphasis — `emphasize`

Where `emphasizeWhen` is a single on/off highlight, `emphasize` flags a cell's
**semantic state** on a warm status ramp. It's an **ordered** list of rules, each
`{ "when": Predicate[], "level": "good" | "warn" | "danger" }`, evaluated against
that row's field values — the **first** matching rule wins. An **empty** `when`
always matches, so a trailing `{ "when": [], "level": … }` is a catch-all tier.
Put the most severe rule first.

```json
{
  "type": "table",
  "source": { "objectType": "invoice" },
  "columns": [
    "name",
    {
      "ref": "status",
      "emphasize": [
        { "when": [{ "field": "status", "op": "eq", "value": "overdue" }], "level": "danger" },
        { "when": [{ "field": "status", "op": "eq", "value": "due_soon" }], "level": "warn" },
        { "when": [], "level": "good" }
      ]
    }
  ]
}
```

Each `level` maps to a **status token**, never a raw color: `good` → success,
`warn` → warning, `danger` → destructive — the same tokens enum chips and
StateMachine colors use. There is no hex / `#f00` escape hatch; a bad `level`
fails decode. Every `when[].field` is checked at declare time against the
source ObjectType. `emphasize` is additive alongside `emphasizeWhen`: when **both**
resolve on the same cell, the semantic `level` wins over the plain highlight.
Rendered by `AutoList` only.

**Known gap.** `when` reuses the FormSpec Predicate grammar, whose ops today are
`eq | in | neq | present` only — there is **no** `gt`/`lt`/`gte`/`lte`. So a
magnitude rule like "color red when `amount > 10000`" is **not yet
expressible**: emphasize must key off an enum/status field (or a precomputed
flag), not a numeric/date threshold comparison.

### Alternate presentation — `present`

`present` shows a value the model **already exposes** in a non-default shape. A
closed literal — `"stat"` or `"progress"` (no free-form escape hatch, no CSS):

- `"stat"` promotes a money / number value to a large, **currency-aware** figure
  (the currency travels with the property — see per-property `currency`).
- `"progress"` renders an **ordered enum** as a step / progress meter (step N of
  M), the step read from the enum's declared `values` order.

```json
{
  "type": "table",
  "source": { "objectType": "deal" },
  "columns": [
    "name",
    { "ref": "amount", "present": "stat" },
    { "ref": "stage", "present": "progress" }
  ]
}
```

A wrong-typed value (`stat` on a non-number, `progress` on a non-enum) falls back
to the default render — never a blank / NaN cell — while a **bad literal**
(`"hero"`, `"timeline"`) fails decode. Rendered by `_presentation.tsx`.

**Bounded-number progress — the object form.** The string `"progress"` shorthand
only knows how to place an **ordered enum** (step N of the enum's declared
`values`); a plain **number** has no such bound, so it rendered bare. To turn a
bounded number — an NPS score (0–10), a utilization %, any `0..max` value — into
the same meter, use the **object form** `{ "kind": "progress", "max": N }`:

```json
{
  "type": "table",
  "source": { "objectType": "account" },
  "columns": [
    "name",
    { "ref": "stage", "present": "progress" },
    { "ref": "nps", "present": { "kind": "progress", "max": 10 } }
  ]
}
```

- The `value / max` ratio fills the meter, labelled **"N of M"**.
- `kind` is a closed literal (`"progress"` today); `max` is **required, positive,
  and finite** — a struct with no valid bound fails decode.
- The object form is **rejected on an enum ref** at `view_specs.declare` — an enum
  already carries its own bound (the `values` order the *string* shorthand reads),
  so a `max` there is redundant/contradictory. Use the string `"progress"` for
  enums, the object form for numbers.

The string shorthand for enums is unchanged (byte-identical to before).

---

## Filters: scalars, ranges, sets, not-equal

Each `filter` entry is one of four shapes: an equality scalar, a range
predicate, a set-membership (`{ in }`) predicate, or a not-equal (`{ neq }`)
predicate.

```json
{
  "type": "table",
  "source": {
    "objectType": "deal",
    "filter": {
      "stage": { "in": ["won", "negotiating"] },
      "owner": { "neq": "house" },
      "amount": { "gte": 1000, "lt": 100000 },
      "created_at": { "gte": "$now-90d" }
    }
  },
  "columns": ["stage", "amount", "created_at"]
}
```

- Equality: `"stage": "won"`; the literal must match the property's declared
  type. Strings accept strings, numbers accept numbers, booleans accept
  `true`/`false`, enums accept only their declared values, and timestamps
  accept ISO timestamp strings (including the relative-date tokens described
  below).
- Range: any of `gt` / `gte` / `lt` / `lte` over a typed bound. The same type
  check applies to every bound; no other keys are allowed (e.g. `between` is
  rejected).
- Set membership: `{ "in": ["won", "negotiating"] }` → SQL `IN (…)`. Members
  are typed equality literals; each is checked against the property's type and
  bound as a parameter (never interpolated). An **empty** `{ "in": [] }` is a
  never-match (`1=0`) at runtime, not an invalid `IN ()`.
- Not-equal: `{ "neq": "house" }` → SQL `<>` the typed bound value.
- Boolean storage compatibility: `true`/`false` are the canonical authoring
  form and are normalized to SQLite `1`/`0` for boolean-mapped columns.
  Numeric `0`/`1` remain accepted as a legacy storage form; prefer the typed
  boolean literals so renderers can preserve the intended value.
- Relative-date token: `$now`, `$now-30d`, `$now+1w` — usable as an equality
  value, a range bound, an `{ in }` member, or a `{ neq }` value. Units are
  `s`, `m`, `h`, `d`, `w` (`m` = **minutes**, NOT months — `$now-1m` is one
  MINUTE; write `$now-30d` for a month-ish window. This exact mistake shipped
  four production charts querying a 60-second window). A wrong
  unit (`$now-1mo`) fails at declare time.
- Relative-date **equality token** (curated): in an **equality** position a
  `:`-prefixed value must be one of a closed set — `:today`, `:yesterday`,
  `:this_week`, `:this_month`, `:last_7d`, `:last_30d` — e.g.
  `{ "created_at": ":this_week" }`. Each resolves server-side to a calendar
  window (`[start, end)`, UTC, weeks start Monday; `:last_7d`/`:last_30d`
  include today). An unknown `:token` is rejected **at decode**. Distinct from
  the `$now±<duration>` range tokens above: the `:token` set is fixed and valid
  only as an **equality** value; a `:token` in a range/`{in}`/`{neq}` position
  is not accepted.

Each op object is shape-pinned: a stray key alongside `in`, `neq`, or a range
key is rejected at declare time, exactly like `between`.

Every filter key must be a real property on the ObjectType.

The same typed-literal rules apply to `source.filter` and to every
`presets[*].filter` on a table. A mismatched literal is rejected during
declaration with the property name and expected type in the diagnostic (for
example, a string supplied for a boolean property is not silently coerced).

### `$principal` / `$actor` — the calling principal's id

A filter value of `"$principal"` (alias `"$actor"`) resolves at render time to
the **current principal's id** — so a "My deals" view can auto-scope
`owner = "$principal"` without the author hard-coding any id.

```json
{
  "type": "table",
  "source": { "objectType": "deal", "filter": { "owner": "$principal" } },
  "columns": ["name", "stage", "amount"]
}
```

Resolution is **server-side, always** — at list/search/aggregate time in the
read path, which already holds the authenticated principal. The renderer sends
the literal token over the wire; there is no client-side resolver and no
drop-when-unresolved branch. The token binds unconditionally to the caller's
principal id, so a caller with no matching row gets **zero rows — never the
unfiltered table**. Every transport (REST/CLI/MCP) resolves it identically. It
is **not** a relative-date token, so the `$now` unit check leaves it untouched;
declare rejects it on a non-string column (`invalid_principal_token_column`).

**`$principal` is relevance, not a security boundary.** It cannot be spoofed
(server-resolved), but it only scopes *this view* — the same caller can run a
raw `objects list` without the filter and see whatever the access policy
allows. When rows must be *invisible* to other principals, declare an access
policy with a `row_filter` (`<col> = :principal.id`, `<col> = :principal.org_id`,
`<col> IN :principal.groups`, or a **literal** — `<col> = 1`,
`<col> = 'published'` — all bound placeholders). Build the "my X" view
with `$principal` for relevance AND the row-filter policy for the boundary;
the policy also covers linked tables the view token cannot reach.

---

## The render context (`$context.*`)

Some blocks need a value that isn't known until render — usually *which*
record we're on. They read it from the render context with the
`$context.<path>` grammar.

- **`detail` record id** — resolved from `$context.<objecttype>.id` first,
  then `$context.id`.
- **Scoped child filter** — a child table inside a composite scopes to its
  parent row with a filter like `{ customer_id: "$context.id" }`. An
  unresolved `$context.*` ref is dropped (never sent as a literal string),
  so a missing context value safely yields no extra filter.
- **`presetParams`** — buttons and panels pre-fill an Action input from the
  row in scope: `{ "deal_id": "$context.id" }`.

```json
{
  "type": "table",
  "source": {
    "objectType": "appointment",
    "filter": { "customer_id": "$context.id" }
  },
  "columns": ["scheduled_at", "status"]
}
```

```
$context = { id: "cus_123", customer: { id: "cus_123" } }
                         |
                         v
   filter { customer_id: "cus_123" }  ->  only this customer's appointments
```
