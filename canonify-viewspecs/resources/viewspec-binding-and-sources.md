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
- `filter` (optional) — equality scalars, range / set / not-equal predicates,
  or the `$principal` token (below).
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
  Leave it off for a single scalar (used by `metric`).
- `filter` (optional) — equality filter over properties.

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
lowercase-snake.

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
One treatment only: a single highlight style.

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
fails decode. `emphasize` is additive alongside `emphasizeWhen`: when **both**
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

- Equality: `"stage": "won"` (a string or number).
- Range: any of `gt` / `gte` / `lt` / `lte` over a string|number bound. No
  other keys are allowed (e.g. `between` is rejected).
- Set membership: `{ "in": ["won", "negotiating"] }` → SQL `IN (…)`. Members
  are equality scalars; each is bound as a parameter (never interpolated). An
  **empty** `{ "in": [] }` is a never-match (`1=0`) at runtime, not an invalid
  `IN ()`.
- Not-equal: `{ "neq": "house" }` → SQL `<>` the bound value.
- Relative-date token: `$now`, `$now-30d`, `$now+1w` — usable as an equality
  value, a range bound, an `{ in }` member, or a `{ neq }` value. Units are
  `s`, `m`, `h`, `d`, `w` (`m` = minutes; **month is not supported**). A wrong
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

Resolution happens client-side in the renderer (the same non-privileged seam
`$context.*` uses), against the principal threaded into the render context. An
**unresolved** token is *dropped* rather than sent as the literal string `"$principal"`
— so a context with no principal safely yields no extra filter. It is **not** a
relative-date token, so the `$now` unit check leaves it untouched.

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
