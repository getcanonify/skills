# Metrics, charts and grouped tables

Aggregate-backed views. Three shapes share one vocabulary: a `metric` (one
scalar tile), a `chart` (drawn — one series, or several via `seriesBy`) and a
grouped `table` (one row per group). All three reuse the same `AggregateSource`,
so a KPI tile, a chart and a summary table can never disagree on what an
aggregate means.

---

## The aggregate vocabulary

| `op` | Property | Notes |
|---|---|---|
| `count` | **none** | Counts rows. A `property` here is rejected. |
| `sum` | required, numeric | Arithmetic over the column. |
| `avg` | required, numeric | Arithmetic over the column. |
| `min` | required | Order statistic — any orderable property. |
| `max` | required | Order statistic — any orderable property. |

`groupBy` present → grouped table or chart. `groupBy` absent → scalar (metric).
All properties (the measure and the `groupBy`) must exist on the ObjectType, and
sum/avg must be numeric — checked at declare time.

### Time buckets — `groupBy: "created_at@month"`

Over a `timestamp` property the useful split is almost never the raw value
(every row has a distinct instant — 10,000 groups of one). So a `groupBy` may
carry a **time-bucket tail**:

```
"groupBy": "created_at@month"
             ^property   ^bucket
```

`bucket` ∈ `day` | `week` | `month` | `quarter` | `year`. Rules:

- Only a **`timestamp`** property can be bucketed. `status@month` is rejected at
  declare (`invalid_group_by_bucket`) — bucketing is a date operation.
- An unknown bucket (`@fortnight`) is rejected by the same check.
- **Empty buckets** are decided at the read, so no view has to guess: a bucket
  with no rows is `0` for `count`/`sum` (absence of activity IS zero) and
  **omitted** for `avg`/`min`/`max` (no observations is not zero — it's nothing
  to average).
- **A sparse span can outgrow the group cap even with few observations** — three
  rows six years apart is 3 observed rows but ~2,200 daily buckets to fill
  between them. When the zero-filled spine would not fit, an **observed row is
  never dropped to make room for a synthetic zero**: the interior gaps are left
  unfilled instead (the result degrades to the raw observed rows, gap = break
  the line), and the read reports it — the caption reads "some groups are
  missing (the group cap was reached)".

The `@bucket` tail is authoring sugar: it never crosses the wire, and the bucket
keys come back in canonical, lexically-sortable forms (`2026-03-15`, `2026-W07`,
`2026-03`, `2026-Q1`, `2026`).

### An explicit window — the newest N buckets

By default a bucketed aggregate spans from the first to the last bucket that has
data (with interior gaps decided as above). When you want a **fixed number of the
most recent periods** — "the last 12 months", ending now, whether or not every
month has rows — ask for an explicit window of the newest N buckets. A window is
measured in buckets, so it **requires** a `group_by_bucket`.

- In a **ViewSpec**, the metric tile's `trend` does this for you — `trend.periods`
  *is* a window of that many buckets (see *Trend* below). ViewSpec authors rarely
  reach for the raw form.
- On the **raw aggregate read**, pass the window directly:

  | Surface | Form |
  |---|---|
  | CLI | `canon objects aggregate <type> --op count --group-by created_at --group-by-bucket month --window-last 12` |
  | REST | `GET /v1/objects/<type>/aggregate?op=count&group_by=created_at&group_by_bucket=month&window_last=12` |
  | MCP | the `objects.aggregate` tool with `window_last: 12` |

The window does two things a plain `filter` range cannot:

- **It fetches only the window.** Only those N buckets are read (the timestamp is
  bounded to the window edges) instead of scanning up to the group cap and
  slicing after the fact.
- **It pads the ends.** For `count`/`sum`, an empty leading or trailing bucket
  *inside the requested window* comes back as a true `0`, so **exactly N points**
  return — even for an ObjectType with only a few periods of history. A
  `count`/`sum` over a requested-but-empty period is genuinely zero; this is the
  end-padding the plain bucketed read cannot do because it never knows the
  intended range. `avg`/`min`/`max` are **not** padded — a period with no rows
  has nothing to average — so for those ops the window is a cap, not a promise of
  N points.

A window wider than the group cap still degrades honestly — the same
cap-truncation caption as above — the window never lifts the cap.

---

## Filtering the aggregate read — the full grammar, on every surface

An analytics `source.filter` (metric, chart, group-by table) speaks the same
grammar as every other source filter — equality scalars, `gt`/`gte`/`lt`/`lte`
ranges, `{ in }`, `{ neq }`, and the relative-date tokens (see
[viewspec-binding-and-sources.md](viewspec-binding-and-sources.md), *Filters*) —
and the render honors all of it: `{ "score": { "gte": 9 } }` on a count metric
counts only the matching rows. (Platforms deployed before 2026-07-24 silently
dropped range predicates on the metric/chart render path and showed the
unfiltered figure — if a dashboard predates that, re-check its range-filtered
tiles.)

On the **raw aggregate read**, the filter rides per surface as:

| Surface | Form |
|---|---|
| CLI | `canon objects aggregate pulse_response --op count --where '{"score":{"gte":9}}'` |
| REST | `GET /v1/objects/<type>/aggregate?op=count&where=<url-encoded json>` — the same `where` JSON grammar the list read speaks. Bare query params (`?stage=won`) remain the equality shorthand; on a same-key collision the structured `where` entry wins. |
| MCP | the `objects.aggregate` tool's `filter` field (a JSON object, same grammar) |

## Metric (KPI tile)

A scalar figure with a label. No `groupBy`.

```json
{
  "type": "metric",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "filter": { "stage": "won" }
  },
  "label": "Revenue (won)",
  "format": "currency"
}
```

```
+----------------------------+
|  Revenue (won)             |
|                            |
|        € 1,240,500         |
+----------------------------+
```

A count metric (no property):

```json
{
  "type": "metric",
  "source": {
    "objectType": "customer",
    "aggregate": { "op": "count" },
    "filter": { "tier": "gold" }
  },
  "label": "Gold customers"
}
```

### Gauge / bar display

A metric can render as a filled meter. `display` is `"scalar"` (default),
`"gauge"`, or `"bar"`; `target` is the fill max — a literal number or a
number-yielding `AggregateSource`. The fill fraction is value/target.

`target` is only meaningful where a meter is drawn: only `"gauge"` and `"bar"`
render one. Declaring `target` on `"scalar"` (or `"hero"`) draws nothing, so it
is **rejected at declare** (`invalid_metric_target`) rather than accepted-and-
ignored — set `display: "gauge"|"bar"`, or drop the `target`.

```json
{
  "type": "metric",
  "source": { "objectType": "seat", "aggregate": { "op": "count" }, "filter": { "assigned": true } },
  "label": "Seats used",
  "display": "gauge",
  "target": { "objectType": "seat", "aggregate": { "op": "count" } }
}
```

### Trend — the stat tile's sparkline

`trend: { property?, bucket, periods }` draws the metric's own measure over
time, under the figure: `periods` points (2–24) bucketed by `bucket`, in grey,
with the current period marked in the accent. `property` defaults to
`created_at` and must be a timestamp.

The sparkline shows **exactly `periods` points**: the trend reads a window of the
newest `periods` buckets (see *An explicit window* above), and for a `count`/`sum`
measure pads empty leading/trailing periods to `0` — so a metric with only a few
months of history still plots the full `periods`-point line rather than a stub.
(For an `avg`/`min`/`max` measure the empty periods have nothing to average, so
the line can be shorter than `periods`.)

```json
{
  "type": "metric",
  "source": { "objectType": "order", "aggregate": { "op": "sum", "property": "total" } },
  "label": "Revenue",
  "format": "currency",
  "compare": { "created_at": "$now-1m.." },
  "goodDirection": "up",
  "trend": { "bucket": "month", "periods": 12 }
}
```

That is the full stat-tile shape: **value + delta + trend**. The delta says what
changed since one prior window; the trend says what the last twelve months
looked like. They read one bucketed aggregate between them and cost one extra
read.

**A trend does not make the metric grouped.** `source.groupBy` is still rejected
on a metric — the tile's figure needs a single number. When you want a real
chart (axes, tooltip, more than 24 points), that is a `chart` block with
`groupBy: "created_at@month"`, below.

### Hero display — the number a dashboard leads with

`display: "hero"` renders the **same figure at display scale** (≥48px): the one
number the view leads with. It is a typographic variant, not a new data shape —
same `source`, same `format`, and the delta badge and `trend` sparkline still
compose below it.

Use it for **exactly one metric per view**. A hero works by being the first
thing the eye lands on; two heroes give the reader no lead, and the view has no
headline. Everything else on the dashboard stays `"scalar"` (the stat tile).

A dashboard led by revenue, with supporting stats beside it:

```json
{
  "type": "composite",
  "children": [
    {
      "type": "metric",
      "source": { "objectType": "order", "aggregate": { "op": "sum", "property": "total" } },
      "label": "Revenue this quarter",
      "format": "currency",
      "display": "hero",
      "compare": { "closed_at": "$now-1y.." },
      "goodDirection": "up"
    },
    {
      "type": "metric",
      "source": { "objectType": "order", "aggregate": { "op": "count" } },
      "label": "Orders",
      "display": "scalar"
    },
    {
      "type": "metric",
      "source": { "objectType": "customer", "aggregate": { "op": "count" }, "filter": { "churned": true } },
      "label": "Churned",
      "display": "scalar",
      "compare": { "churned_at": "$now-1q.." },
      "goodDirection": "down"
    }
  ]
}
```

The hero above reads `$1.3M` at 48px with a green `+12%` badge under it
(`goodDirection: "up"` — a rise in revenue is good news); the churn tile beside
it tones its own `+8%` red from `goodDirection: "down"`. Values ≥10,000 are
compacted (`1.3M`, `12.9K`) — a hero is a headline, not a ledger.

A hero renders **no meter**: it is a figure, not a gauge, so a `target` on a
hero is **rejected at declare** (`invalid_metric_target`) — only
`display: "gauge"|"bar"` draws a fill. Pair `goodDirection` with `compare` on a hero (the delta badge
is the only signal a hero can tone) — `goodDirection` on a hero with neither is
rejected at declare as inert.

More than one hero in a view is **not an error** — it declares and renders.
`view_specs.preview` reports it as a `multiple_hero_metrics` advisory naming
each offending block, so the authoring loop tells you without blocking you.

### Record-scoped metric (one hop)

On a record-scoped view (a `detail`), a metric can aggregate **the current
record's related rows over ONE link** instead of a whole ObjectType:

```json
{
  "type": "metric",
  "source": { "link": "orders", "aggregate": { "op": "sum", "property": "total" } },
  "label": "Lifetime spend",
  "format": "currency"
}
```

`link` names a SINGLE link — a dotted two-hop value (`customer.orders`) is
rejected at decode. At declare time the link must resolve on the record's
ObjectType, and a `sum`/`avg` property must exist and be numeric on the LINK
TARGET.

---

## Grouped summary table

Same source, plus `groupBy` — one row per group. The `columns` name the group
key and the measure.

```json
{
  "type": "table",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "groupBy": "stage"
  },
  "columns": ["stage", "amount"]
}
```

```
+-----------------------------+
|  Pipeline by stage          |
+-----------+-----------------+
|  Stage    |  Amount (Σ)     |
+-----------+-----------------+
|  lead     |   € 210,000     |
|  proposal |   € 540,000     |
|  won      |   € 1,240,500   |
|  lost     |   € 340,000     |
+-----------+-----------------+
```

A grouped count (one row per group, counting rows):

```json
{
  "type": "table",
  "source": {
    "objectType": "appointment",
    "aggregate": { "op": "count" },
    "groupBy": "status"
  },
  "columns": ["status", "count"]
}
```

---

## Chart (bar / line)

The same `AggregateSource`, **drawn**: `groupBy` becomes the dimension axis and
`aggregate` the measure axis. A chart REQUIRES `groupBy` — a source without one
is a single scalar, so declare rejects it (`invalid_chart_group_by`) and points
you at `metric`.

```json
{
  "type": "chart",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "count" },
    "groupBy": "stage"
  },
  "title": "Deals by stage"
}
```

```
   Deals by stage                        [ Show table ]
   40 |  ██
      |  ██    ██
   20 |  ██    ██    ██
      |  ██    ██    ██    ██
    0 +--------------------------
        lead  prop.  won   lost
```

### `kind` is inferred — set it only to override

| Source | Inferred `kind` | Why |
|---|---|---|
| categorical `groupBy` (`"stage"`) | `bar` | magnitude by category |
| time-bucketed `groupBy` (`"created_at@month"`) | `line` | a trend over time |

So the common cases need no `kind` at all — the source already says which shape
the data has. Set `kind` explicitly (`"bar"` \| `"line"`) only when you want the
other one, e.g. monthly revenue as columns:

```json
{
  "type": "chart",
  "kind": "line",
  "format": "currency",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "groupBy": "created_at@month",
    "filter": { "stage": "won" }
  },
  "title": "Won revenue by month"
}
```

An unknown `kind` (`"pie"`, `"scatter"`) is rejected at decode — it never
silently degrades to a bar.

### Fields

| Field | Required | Notes |
|---|---|---|
| `type` | yes | `"chart"` |
| `source` | yes | An `AggregateSource` **with** `groupBy`. |
| `kind` | no | `"bar"` \| `"line"`. Absent → inferred (see above). |
| `stacked` | no | `true` → stacked bars. Requires `seriesBy`, `bar`, and an additive op (see below). |
| `format` | no | `"currency"` \| `"percent"` \| `"date"` — the **value axis** + tooltip. |
| `title` | no | Section heading above the chart. |
| `showWhen` | no | Standard render gate. |

### What the renderer does for you

You declare the data; these are not knobs:

- **Currency travels with the property.** `format: "currency"` names no currency
  code — the ISO-4217 code is read from the aggregated property's own declared
  `currency`, so a sum of a DKK column reads as DKK on the axis, in the tooltip
  AND in the table view. A property with no declared currency renders a bare
  grouped number: there is no default, because a wrong currency is worse than
  none.
- **Sort is inferred from the catalog.** If the `groupBy` property is an enum
  that declares its `values`, the dimension is ORDINAL and bars are drawn in
  that declared order (a pipeline stays in pipeline order). Otherwise it is
  NOMINAL and bars sort by value, descending. A time series is chronological.
- **High cardinality folds.** Beyond 12 categories the tail folds into a single
  **Other** mark — summed for `count`/`sum`, min/max-composed for `min`/`max`.
  An `avg` tail is **never** folded (the average of averages is not the average),
  so the chart truncates and says so. A time series never folds.
- **Long labels flip the bars horizontal.** No rotated or clipped text.
- **A table view is always one click away** — the same grouped aggregate as the
  `table` block, so every value stays reachable without hovering.
- **One series, one hue, no legend.** With no `seriesBy` a chart plots one
  measure over one dimension and the title names it — a one-swatch legend would
  just restate the title. Two MEASURES is still two charts (never a second
  y-axis); `seriesBy` splits ONE measure, which is a different thing.

### `seriesBy` — splitting one measure into series

`seriesBy` adds a **second, categorical** dimension: `groupBy` stays the axis,
and each distinct `seriesBy` value becomes its own series. "Revenue by month
**split by** product line" is `groupBy: "created_at@month"` +
`seriesBy: "product_line"`.

```json
{
  "type": "chart",
  "format": "currency",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "groupBy": "created_at@month",
    "seriesBy": "product_line"
  },
  "title": "Revenue by month, by product line"
}
```

`seriesBy` takes a **bare property name** — no `@bucket` tail. A series is a
categorical split; the time bucket belongs on `groupBy`, the dimension it
buckets along. `product_line@month` is rejected at decode. It also **requires**
`groupBy` and must name a **different** property than `groupBy`
(`invalid_series_by`) — splitting a dimension by itself yields one series per
group with nothing to compare.

### `stacked` — only where the total means something

Absent → **grouped** bars (series side by side; the reader compares series).
`stacked: true` → **stacked** bars (segments compose one column; the reader
compares TOTALS).

A stack's total height is a number the reader reads, so declare rejects
(`invalid_chart_stacked`) any stack whose total would be meaningless:

| Rejected | Why |
|---|---|
| `stacked` with `avg` / `min` / `max` | Only `count`/`sum` are additive over a partition. The sum of per-series averages is not an average of anything. |
| `stacked` with no `seriesBy` | A one-segment stack is just a bar. |
| `stacked` on a `line` | Stacking is a bar treatment. (Note the `kind` may be **inferred** `line` from a bucketed `groupBy` — set `kind: "bar"` to stack a time series.) |

### What the renderer does for a multi-series chart

- **A legend, always, naming every series in TEXT.** Identity is carried by the
  label, not the swatch — never "the blue one". With ≤4 lines each series is
  also labelled at the end of its own line.
- **Fixed hue order, never cycled.** Series take `--chart-1..5` in sequence, in
  a stable identity order (the property's declared enum order if it has one,
  else alphabetical) — never in magnitude order, so a series' colour does not
  move when its number moves.
- **Past 5 series the tail folds into Other**, using the same op-correct
  combiner: summed for `count`/`sum`, min/max-composed for `min`/`max`, and an
  `avg` tail is **never** folded — it is dropped and the caption says so.
- **The tooltip lists every series** at the hovered point, in one tooltip.
- **The table view gains a series column**, so it shows the same split the
  chart does rather than totals across series.
- **Caps are never silent.** If the server's group cap drops whole series the
  caption says so — and every series it DOES show is complete.

---

## KPI dashboard composite

Stack a row of metrics over a chart or grouped table. A `composite` whose
children are several `metric` tiles plus a `chart` is the canonical dashboard.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "metric",
      "source": { "objectType": "deal", "aggregate": { "op": "count" } },
      "label": "Open deals"
    },
    {
      "type": "metric",
      "source": {
        "objectType": "deal",
        "aggregate": { "op": "sum", "property": "amount" },
        "filter": { "stage": "won" }
      },
      "label": "Revenue (won)",
      "format": "currency"
    },
    {
      "type": "metric",
      "source": {
        "objectType": "deal",
        "aggregate": { "op": "avg", "property": "amount" }
      },
      "label": "Avg deal size",
      "format": "currency"
    },
    {
      "type": "table",
      "source": {
        "objectType": "deal",
        "aggregate": { "op": "sum", "property": "amount" },
        "groupBy": "stage"
      },
      "columns": ["stage", "amount"]
    }
  ]
}
```

```
+----------------+  +------------------+  +----------------+
| Open deals     |  | Revenue (won)    |  | Avg deal size  |   <- metric row
|      42        |  |   € 1,240,500    |  |    € 29,500    |
+----------------+  +------------------+  +----------------+
+--------------------------------------------------------+
|  Pipeline by stage                                     |   <- grouped table
|  Stage     |  Amount (Σ)                               |
|  lead      |   € 210,000                               |
|  proposal  |   € 540,000                               |
|  won       |   € 1,240,500                             |
+--------------------------------------------------------+
```
