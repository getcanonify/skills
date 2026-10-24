# Binding and sources

The grammar for *what* a block reads: sources, aggregates, filters, columns,
formats and the render context. Everything is checked against the live
registry at `view_specs.declare` / `catalog.apply` time.

## Record source — "show these rows"

`{ objectType, filter?, sort? }` — used by table, detail, board, calendar and queue.

```json
{
  "type": "table",
  "source": { "objectType": "customer", "filter": { "tier": "gold", "created_at": { "gte": "$now-30d" } }, "sort": "-created_at" },
  "columns": ["name", "email", "tier"]
}
```

- `sort` — one own property, `-` prefix for descending. A queue sorts only by
  its `order` (a `source.sort` on a queue is refused at decode).
- `page_size` lives on the `table` / `queue` block, not on `source`: 1–200,
  default 50, the size of the first page and of each "Load more". It never caps
  the total.

## Aggregate source — "reduce these rows to a figure"

`{ objectType, aggregate, groupBy?, seriesBy?, expand?, filter? }` — shared by
`metric`, `chart` and a grouped `table`, so the three can never disagree on
what a measure means. A `table` with `aggregate` is a grouped summary; without
it, a record list.

| `aggregate.op` | `property` | |
|---|---|---|
| `count` | none (a property is refused) | counts rows |
| `sum`, `avg` | required, numeric | |
| `max` | required, any orderable | |

`groupBy` present → one row / bar per group; absent → one scalar (a metric).
`seriesBy` is a second, categorical split: it needs `groupBy`, must name a
different path (`invalid_series_by`) and never takes a `@bucket`.

**Every result names its unit.** A `sum`/`avg`/`max` result carries the
measured property's type and currency (`"unit": { "type": "money", "currency":
"DKK" }`) in the units a record read returns (integer minor units for money);
`count` has none. So money and currency-declared numbers render as amounts with
no `format`, and a link aggregate column (`analyses.sum:revenue`) takes the link
target property's unit the same way.

### Time buckets — `groupBy: "created_at@month"`

A `timestamp` or `date` `groupBy` may carry a bucket tail: `@day`, `@week`,
`@month`, `@quarter`, `@year`. Bucketing any other type, or an unknown bucket,
is `invalid_group_by_bucket`. `created_at` / `updated_at` are typed `timestamp`
on every ObjectType without a refinement; another column needs `type:
"timestamp"` or `"date"` first.

- **Buckets follow the org's calendar** (`org.update { "timezone" }`; UTC when
  unset): an analysis released 00:30 Copenhagen time on 1 April counts in April.
  Window edges, the zero-filled spine, the partial bucket and `prior_to_date`
  read the same clock, daylight saving included.
- **Empty buckets:** `0` for `count`/`sum`; omitted for `avg`/`max`.
- **The group cap.** When a zero-filled spine would not fit (three rows six
  years apart is ~2,200 days), interior gaps stay unfilled rather than drop an
  observed row, and the caption says some groups are missing.
- The tail never crosses the wire (it becomes `group_by` + `group_by_bucket`);
  keys come back as `2026-03-15`, `2026-W07`, `2026-03`, `2026-Q1`, `2026`.

### `window` and `compare` — the aggregate envelope

`metric`, `chart` and grouped `table` all carry `window` and `compare`, with
one rule set:

- **`window: { last: N }`** — only the newest N buckets (N ≤ 4000), or
  **`{ range: ":next_7d" }`** the buckets a calendar token covers (forward too).
  It pads its ends: `count`/`sum` return every bucket, an empty one a true `0`;
  `avg`/`max` are not padded. Never lifts the group cap; needs a `@bucket` spine
  (`invalid_aggregate_window`).
- **`compare: { offset?: { period, periods }, rolling?: { buckets } }`** — at
  least one. `offset.period` is `month` or `year`, `periods` 1–5 (a prior-period
  copy of the same measure); `rolling.buckets` 2–12 (a trailing mean).

`compare` is refused with `invalid_aggregate_compare` without a `@bucket`
spine, with `seriesBy`, over `avg`/`max`, or with a `source.filter` on the
spine's own property (it would cut the widened scan the layers come from —
scope a compared view with `window`). An offset the spine cannot address is
`invalid_aggregate_compare_offset`:

| Spine bucket | Admitted `offset.period` |
|---|---|
| `day` | `month`, `year` |
| `week` | — (ISO weeks do not survive a year shift) |
| `month` | `month`, `year` |
| `quarter` | `year` |
| `year` | `year` |

**The response** (raw read and views alike) adds per group: `prior` — the
shifted bucket (`0` is a fact; `null` only when the calendar lacks the date,
e.g. Feb 29); `rolling` — `null` at the partial bucket; `partial: true` on the
bucket containing now, on **every** bucketed read whether or not compare was
asked; and, on the partial bucket with an offset, `prior_to_date` — the same
elapsed span of the shifted bucket (on 16 October a month spine reports
September 1–16). Its `from` / `until` carry the org's UTC offset
(`2026-09-01T00:00:00.000+02:00`; `Z` only for a UTC org).

```json
{
  "unit": { "type": "money", "currency": "DKK" },
  "groups": [
    { "group": { "created_at": "2026-01" }, "value": 15000, "prior": 12000, "rolling": 13500 },
    { "group": { "created_at": "2026-02" }, "value": 9000, "prior": 11000,
      "prior_to_date": { "value": 6500, "from": "2025-02-01T00:00:00.000Z", "until": "2025-02-16T11:30:00.000Z" },
      "rolling": null, "partial": true }
  ]
}
```

### Filtering the aggregate read — the same grammar on every surface

An aggregate `source.filter` speaks the full filter grammar below, and the raw
read accepts the same window, compare and filter as parameters:

| Surface | Form |
|---|---|
| CLI | `canon objects aggregate analysis --op sum --property fee --group-by created_at --group-by-bucket month --window-last 12 --compare-period year --compare-periods 1 --rolling 3 --where '{"status":"queued"}'` (`--window-range :next_7d` for a calendar window; `--period-offset -1` moves every `:this_week`-style filter token to the window before) |
| REST | `GET /v1/objects/<type>/aggregate?op=count&group_by=created_at&group_by_bucket=month&window_last=12&compare_period=year&rolling=3&where=<url-encoded json>`; a bare param (`?stage=won`) is equality shorthand, and `where` wins a same-key collision |
| MCP | the `objects.aggregate` tool: `window_last` / `window_range`, `period_offset`, `compare_period`, `compare_periods`, `rolling`, `filter`, `expand` |

The raw read's `compare_period` admits `day` · `week` · `month` · `quarter` ·
`year` (`compare_periods` defaults to 1): on a `day` spine `day`, `week`,
`month`, `year`; on `week` only `week`; on `month` also `quarter`; on `quarter`
`quarter`, `year`. A mismatch is a `VALIDATION` 400 naming the admitted set.

### Link paths, `expand`, and derived measures

Paths in `groupBy`, `seriesBy`, filter keys and measure factors take at most
two to-one link hops before the terminal property.
`expand` names the one path whose final hop is to-many; it is the only place
fan-out is allowed. Earlier hops are to-one (LEFT JOIN); the expanded hop is an
INNER JOIN, so a missing component contributes no row. An `expand` nothing
uses is `expand_unused`.
Once a source expands, use `aggregate: { "op": "count" }` or a labeled
`aggregate.of` measure; a plain `sum` is rejected with `expand_requires_measure`.
Summing a root property after a fan-out would multiply it.

`aggregate.of` is a 1–3-factor measure; each factor is numeric and at most one
factor may be a `lookup`. It is a product summed over the rows (`op: "sum"`
only, never with `property`), and a table or chart must give it an
`aggregate.label` (`measure_label_required`). A `lookup` is an equality-only
correlated read `{ objectType, match, property, default }`: `match` has 1–3
type-checked keys, the property is numeric, `default` is finite, several hits
resolve to the maximum, and the lookup type is policy-gated like a join.

```json
{ "type": "table",
  "title": "Portion plan",
  "source": {
    "objectType": "order_line",
    "expand": "product.components",
    "filter": { "order.location_id": "$context.location_id", "order.delivery_date": "$context.delivery_date", "order.status": "locked" },
    "groupBy": "product.components.slot",
    "aggregate": {
      "op": "sum", "label": "Portions",
      "of": [
        "quantity",
        "product.components.default_ratio",
        { "lookup": { "objectType": "customer_component_ratio",
                      "match": { "customer_id": "order.customer_id", "component_id": "product.components.component_id" },
                      "property": "ratio", "default": 1 } } ] }
  },
  "columns": ["product.components.slot"] }
```

At location `L1` on `2026-09-07`: P1 has C1 (`salad`, `0.5`) and C2 (`hot`,
`1.0`); P2 has C2 (`hot`, `0.8`); customer A overrides C1 to `2.0`. Locked lines
A/P1 ×10, B/P1 ×4, B/P2 ×5; an open line and an L2 order are filtered out.
Therefore PortionPlan fixture math:
salad = 10×0.5×2.0 + 4×0.5×1 = **12** and hot = 10×1.0×1 + 4×1.0×1 + 5×0.8×1 = **18**. The `$context.*` values are
resolved by the read, never interpolated into SQL.

## Filters

`source.filter`, table `presets` and composite filters are the filter
spelling of the [one condition grammar](../../canonify-objects-actions/resources/actions-handler-patterns.md#conditions--one-grammar-for-every-surface) (≤ 1000 `in` members) —
"open, or closed in the last 30 days" is
`{ "$or": [{ "status": "open" }, { "closed_at": { "gte": "$now-30d" } }] }`.

Path filters use the same closed
grammar as columns: `property`, `link.property`, or `link.link.property`;
every non-terminal segment must resolve to a declared to-one link. A
to-many link is never legal in a filter key; declaration fails with
`link_path_to_many`. Unknown segments are `unknown_column_ref`.
Every traversed ObjectType is policy-gated exactly like a direct read of it.
Its row filter and column projection apply at the join, and a principal who
cannot read it gets a loud failure, never a widened result.

- **Relative dates.** `$now`, `$now±<n><unit>` with units `s m h d w mo y` —
  `m` is **minutes**, `mo` months (`$now-1m` is one minute; write `$now-1mo`).
  Usable as an equality or a range bound; a wrong unit fails at declare.
  `$today` / `$today±<n>d` name an org calendar day: `{ "due": { "lte": "$today" } }`
  is due on or before today.
- **Calendar tokens.** `:today`, `:yesterday`,
  `:tomorrow`, `:this_week`, `:next_week`, `:this_month`, `:last_7d`,
  `:last_30d`, `:next_7d`, `:next_30d` (7 / 30 days ending or starting today) — each a `[start, end)` window on the org's calendar (UTC when
  unset), weeks start Monday. Any other `:token` fails at decode. A window (and
  `$today`) compares as conditions do: equality is inside it, `gte` from its
  start, `lt` before it, `lte` before its end, `gt` after it.
- **Unresolved `$`-values refuse.** A filter value starting `$` must be a
  relative time, `$context.<path>` (the record in view) or the invoker
  (`$principal.<path>`, `$actor.<attribute>`, `$actor_id`); anything else
  (`$tomorow`) fails at declare with `unresolved_condition_operand` rather than
  compare as text.
- **`$as_of`** — on a temporal (effective-dated) type, `{ "$as_of": "$now" }`
  keeps the periods in effect on that date.

### `$principal` — "my records"

`"$principal"` resolves server-side, on every transport, to the viewer's id:
`{ "owner": "$principal" }` is a "My deals" view, and a viewer with no matching
row sees zero rows, never the whole table. It needs a string column
(`invalid_principal_token_column`). It is relevance, not a boundary: the same
caller can list the type unfiltered. When rows must be invisible, declare an
access policy `row_filter` as well
([access-policies.md](../../canonify-objects-actions/resources/access-policies.md)).

## Column references

| Form | Meaning |
|---|---|
| `"email"` | A property of the source type |
| `"customer.tier"` | Up to two to-one link hops, then a property |
| `"appointments.count"` | Row count over a to-many link |
| `"orders.sum:amount"` | `sum` / `avg` / `max` of a link target property (numeric for sum/avg) |
| `{ "ref", "label"?, "sortable"?, "format"?, "emphasize"?, "of"? }` | The object form; `of` (table column or header metric: a number or money property of the same row, one unit) draws `ref` as its share: a bar and the percent, warn from 80 %, danger over the limit (`invalid_progress_of`) |

A column with no `label` reads the property's declared `label` (a traversal
column reads the FK column's label, else the target's `entityKind`). A bad ref
is `unknown_column_ref`. **`sortable: true` works on the type's own
properties** — mapped, derived or lookup — **and its declared rollups**; on a
dotted ref (`customer.tier`, `orders.sum:amount`) it is
`unsupported_sortable_column_ref`: declare a lookup or a rollup and sort by
that. For a row-scoped reader, show a parent's field as a link ref or a
lookup, never a derived property ([why](../../canonify-objects-actions/resources/access-policies.md#derived-properties-under-access)).

### `format`

`currency` needs a number or money property, `percent` a number it reads as a
ratio (`0.055` → `5.5%`), `date` a date or timestamp (`invalid_column_format`). `format: "currency"` is a rendering hint
only — the denomination is the property's declared `currency`, so money needs
no `format`. `format: "date"` shows the calendar day only: a real instant shows
the day it falls on the org's clock; a calendar date (a `date` property, a bare
`2026-10-15`, or a timestamp at exactly midnight UTC) shows its stored day for
every reader. A plain timestamp column already shows date and time.

### `emphasize` — semantic emphasis

An ordered list `[{ when, level }]`, first match wins; an empty `when` is a
catch-all. `level` is `good` / `warn` / `danger`, mapped to the success /
warning / destructive status tokens (no raw colours). `when` is the
[condition grammar](../../canonify-objects-actions/resources/actions-handler-patterns.md#conditions--one-grammar-for-every-surface) on the source type's own
properties and rollups — `{ "field": "open_count", "op": "gt", "value": 0 }`
is a threshold; a `$principal.*` field is refused (a rule reads the row). How
it paints, and where else the same property inherits it: declare it once —
[what reads first](viewspec-block-types.md#what-reads-first).

```json
{
  "type": "table",
  "source": { "objectType": "invoice" },
  "columns": [
    "name",
    { "ref": "amount", "format": "currency" },
    { "ref": "status", "emphasize": [
      { "when": [{ "field": "status", "op": "eq", "value": "overdue" }], "level": "danger" },
      { "when": [], "level": "good" } ] }
  ]
}
```

## The render context — `$context.*` and `$row.*`

- A `detail` resolves its record from `$context.<objecttype>.id`, then
  `$context.id`.
- A child of a record-scoped composite scopes with
  `{ "customer_id": "$context.id" }`; an unresolved `$context.*` ref is dropped
  (never sent as a literal), so it adds no filter.
- `presetParams` on buttons, panels and row actions take literals and
  `$context.*` values; a row action's `$row.<field>` binds the row's own
  property and is checked against the source type at declare time.
- The context is a typed render input, resolved before the read and subject to
  the same path, literal and policy checks as any filter value.
