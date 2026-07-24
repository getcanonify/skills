# The 10 block types

A ViewSpec is a discriminated union — one JSON object whose `type` field
picks the block. The renderer dispatches on `type`. Every block also accepts
an optional `showWhen` (see forms-and-actions). All shapes are **closed**:
unknown fields are rejected at declare time.

The ten types: `table`, `detail`, `panel`, `button`, `composite`,
`metric`, `chart`, `file`, `board`, `queue`.

---

## 1. `table` — a list

One row per record (object-backed) or one row per group (aggregate-backed).
`columns` must have at least one entry.

```json
{
  "type": "table",
  "source": {
    "objectType": "customer",
    "filter": { "tier": "gold" },
    "sort": "name"
  },
  "columns": [
    "name",
    "email",
    { "ref": "loyalty_tier", "label": "Tier", "sortable": true },
    "appointments.count"
  ]
}
```

```
+----------------------------------------------------------+
|  Customers (tier = gold)                    [ filter ▾ ] |
+----------------+---------------------+-------+-----------+
|  Name          |  Email              | Tier  | Appts (#) |
+----------------+---------------------+-------+-----------+
|  Acme Corp     |  ops@acme.test      | Gold  |    12     |
|  Beta Ltd      |  hi@beta.test       | Gold  |     7     |
+----------------+---------------------+-------+-----------+
```

**Filter presets.** A `table` may declare `presets` — curated one-tap filter
chips. Each is `{ "label": "<caption>", "filter": { … } }`, where `filter`
reuses the same source-filter grammar (equality scalar, `:this_week` calendar
token, range, `{in}`, `{neq}`). Tapping a chip applies its filter; it is a
DECLARED set, not user-saved state. An empty `label` is rejected at decode.

```json
{
  "type": "table",
  "source": { "objectType": "ticket" },
  "columns": ["subject", "status", "created_at"],
  "presets": [
    { "label": "Open", "filter": { "status": { "in": ["open", "pending"] } } },
    { "label": "This week", "filter": { "created_at": ":this_week" } }
  ]
}
```

**Per-row actions.** `rowActions` replaces the inferred per-row action set with a
curated list. Each entry is `{ action, label?, presetParams?, default?, showWhen? }`
(a bare string `"deal.advance"` is sugar for `{ "action": "deal.advance" }`).
`presetParams` binds action inputs from the current row via `$row.<field>` tokens
(the subject id auto-binds). `default: true` marks the action that stays inline
when the row's actions overflow into an overflow menu; the renderer uses the first
flagged action, else the first declared. Each `action` is validated at declare
time — an unknown action fails with `unknown_action`.

```json
{
  "type": "table",
  "source": { "objectType": "deal" },
  "columns": ["name", "stage", "amount"],
  "rowActions": [
    { "action": "deal.advance", "label": "Advance", "default": true },
    { "action": "deal.archive", "presetParams": { "reason": "$row.stage" } }
  ]
}
```

**Page size.** `page_size` (optional, integer 1–200; default 50) sets the fetch
size for the first page and each "Load more" page. It never caps total rows.

**Row grouping.** `rowGroups` sections a table's per-record rows into named,
**collapsible** groups keyed by one property's value. Every matching row is kept
(rows are unchanged) — grouping only inserts a collapsible header (label + count)
above each run of rows. The shape is `{ by, groups?, defaultCollapsed? }`:

- `by` (required) — the property whose value sections the rows. It must exist on
  the ObjectType (checked at declare — `invalid_row_groups_by`).
- `groups` (optional) — explicit `{ value, label }` entries giving a label and
  **order** to known values. Sections render in that order first; values not
  listed render after, auto-labeled by their raw value. A declared group renders
  even when empty.
- `defaultCollapsed` (optional) — absent / `false` ⇒ sections start **expanded**
  (they are always collapsible); `true` opts into collapsed-by-default.

```json
{
  "type": "table",
  "source": { "objectType": "task" },
  "columns": ["title", "priority", "assignee"],
  "rowGroups": {
    "by": "priority",
    "groups": [
      { "value": "high", "label": "High priority" },
      { "value": "low", "label": "Low priority" }
    ],
    "defaultCollapsed": false
  }
}
```

`rowGroups` is **presentation-level sectioning**, deliberately distinct from the
three other grouping surfaces — pick by intent:

| Knob | Groups | Effect |
|---|---|---|
| `rowGroups.by` | a `table`'s per-record rows | collapsible label + count sections; every row kept |
| `AggregateSource.groupBy` | data | collapses rows to ONE measure row per group |
| `board.groupBy` | a `board`'s cards | one kanban COLUMN per enum value |
| `queue.buckets` | a `queue`'s rows | a FIXED, renderer-owned date scheme (Overdue / Today / …) |

It is also **not** the `AggregateSource` time-bucket tail (`created_at@month`,
see analytics), which shapes DATA over a date rather than sectioning rows. `rowGroups` on an **aggregate-backed** table is rejected at declare
(`row_groups_on_aggregate`) — that source already collapses to one row per group,
so use its `groupBy`. Rendered by `_row-groups.tsx`.

---

## 2. `detail` — one record's properties

A labelled property grid for a single record. The record id comes from the
render context (`$context.<objecttype>.id`, falling back to `$context.id`).
`properties` is optional — omit it to use the ObjectType's default grid.

```json
{
  "type": "detail",
  "source": { "objectType": "customer" },
  "properties": [
    "name",
    "email",
    { "ref": "tier", "label": "Loyalty tier" }
  ]
}
```

```
+--------------------------------------+
|  Customer                            |
+--------------------------------------+
|  Name          Acme Corp             |
|  Email         ops@acme.test         |
|  Loyalty tier  Gold                  |
+--------------------------------------+
```

**Record header.** A `detail` may declare an optional `header` — a
title/subtitle/status-chip/avatar band above the property grid, each a field
ref:

```json
{
  "type": "detail",
  "source": { "objectType": "customer" },
  "header": { "title": "name", "subtitle": "email", "status": "tier", "avatar": "logo_url" }
}
```

Every field is independently optional — omit any and `AutoDetail` falls back to
its inferred behavior for that slot. `status` should name an **enum** property;
`avatar` should name an **image-URL** property.

**`subtitle` accepts a single ref _or_ a template.** The plain form names one
property (`"email"`) or a **link traversal** (`"organization.cvr"` — follow the
`organization` link, read its `cvr`). To compose a richer line, use a **template**
carrying `{ref}` placeholders:

```json
{
  "type": "detail",
  "source": { "objectType": "customer" },
  "header": {
    "title": "name",
    "subtitle": "{lifecycle_state} · Account manager {account_owner} · CVR {organization.cvr}"
  }
}
```

Grammar rules:

- Each `{ref}` is a **property or a link traversal**, resolved exactly the way a
  grid column ref (or a single-ref subtitle) is. An enum placeholder renders its
  `valueLabel` when one is declared.
- A **null / missing** placeholder renders an **em-dash (`—`) in place**, keeping
  the surrounding separators intact — a predictable empty-value rule, so a partly
  populated record still reads cleanly instead of collapsing punctuation.
- **Unknown refs are rejected at declare time** (`unknown_column_ref` at
  `view_specs.declare` / `catalog.apply`) — a typo fails the apply with a clear
  error rather than silently rendering an em-dash forever.
- A **plain, brace-less** subtitle is unchanged — it is not newly validated, and
  behaves byte-for-byte as before. The `{…}` grammar is the same one the
  display-title `template` strategy uses, broadened to allow dots (traversals).

The header may also carry `metrics` — a declared **metric strip**, a row of
header stats using the same `ColumnRef` grammar the grid columns use (a bare
`"total"`, or `{ "ref": "balance_due", "label": "Due", "format": "currency" }`):

```json
{
  "type": "detail",
  "source": { "objectType": "invoice" },
  "header": {
    "title": "number",
    "status": "status",
    "metrics": ["total", { "ref": "balance_due", "format": "currency" }]
  }
}
```

`metrics` **formalizes** the auto-derivation (tick 14b): omit it and a
currency-declared money field still auto-becomes a header metric strip; declare
it to name the exact set and relabel / format each. A malformed ref fails decode,
so an unrenderable metric can't ship.

---

## 3. `panel` — an action form

Hosts a form generated from an Action's input schema. `actionForm.action`
names the Action; `submitLabel` and `presetParams` are optional.

```json
{
  "type": "panel",
  "actionForm": {
    "action": "customer.create",
    "submitLabel": "Add customer",
    "presetParams": { "tier": "standard" }
  }
}
```

```
+--------------------------------------+
|  New customer                        |
+--------------------------------------+
|  Name    [____________________]      |
|  Email   [____________________]      |
|  Tier    [ standard          ▾]      |
|                       [ Add customer ]|
+--------------------------------------+
```

---

## 4. `button` — a one-tap action

Fires an Action. Use `presetParams` to supply its input, `label` to override
the default caption.

```json
{
  "type": "button",
  "action": "deal.advance",
  "label": "Advance stage",
  "presetParams": { "deal_id": "$context.id" }
}
```

```
+------------------+
|  Advance stage   |
+------------------+
```

---

## 5. `composite` — a container

Stacks other blocks top-to-bottom. `children` is a non-empty array of full
ViewSpecs, so composites nest.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "metric",
      "source": { "objectType": "customer", "aggregate": { "op": "count" } },
      "label": "Total customers"
    },
    {
      "type": "table",
      "source": { "objectType": "customer" },
      "columns": ["name", "email"]
    }
  ]
}
```

```
+--------------------------------------+
|  +--------------------------------+  |
|  | Total customers          128   |  |   <- metric child
|  +--------------------------------+  |
|  +--------------------------------+  |
|  | Name        | Email           |  |   <- table child
|  | Acme Corp   | ops@acme.test   |  |
|  +--------------------------------+  |
+--------------------------------------+
```

**Optional fields.** `layout` (`"stack"` default | `"grid"` | `"tabs"`) arranges
the children; `filters` adds a reader-adjustable filter row above them (each entry
is `{ "property", "kind": "date-range" | "dimension" }`) that scopes every child
to the same slice so the figures agree; `title` is section chrome. Both are
covered with worked examples in
[viewspec-composition-patterns.md](viewspec-composition-patterns.md).

---

## 6. `metric` — a scalar KPI tile

An `AggregateSource` used *without* `groupBy` — one number. `label` is
required; `format` is an optional render hint. `display` (`"scalar"` default,
`"gauge"`, `"bar"`, `"hero"`) picks the presentation: `"gauge"`/`"bar"` with an
optional `target` (a literal number or a number-yielding aggregate) render the
value as a filled meter, and `"hero"` renders it at display scale (≥48px) as
the one number a view leads with — one per view. `trend` adds a time-series
sparkline under the figure. The source may instead be a
record-scoped one-hop aggregate `{ link, aggregate }` (see
[viewspec-analytics](viewspec-analytics.md), which has a dashboard-lead
example).

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

### `goodDirection` — is UP good news?

`goodDirection` (`"up"` | `"down"`) declares which direction is **favourable**.
Without it a metric has no polarity, so the tile must stay neutral: `+5%` looks
identical on revenue and on churn. Declare it and two things start carrying
meaning:

- **The delta badge tones** by *direction × polarity*. A rise on an `"up"`
  metric and a fall on a `"down"` metric both read positive; the opposites read
  negative. A `0%` change stays neutral. **The `+`/`-` sign prefix is always
  kept** — the tone is a redundant second channel, never the only one.
- **A meter fill carries severity.** On a `"down"` metric (a quota, a capacity,
  an error budget — filling up is *bad*) the fill ramps accent → warning at 75%
  of `target` → danger at 90% and over. On an `"up"` metric the fill is progress
  toward a goal, so it **stays accent** at every level: a bar at 95% of a revenue
  target is the best case, not an alarm. The unfilled track is a lighter step of
  the fill's own ramp, so the state reads across the whole bar.

```json
{
  "type": "metric",
  "source": {
    "objectType": "customer",
    "aggregate": { "op": "count" },
    "filter": { "status": "churned" }
  },
  "label": "Churned this month",
  "compare": { "created_at": "$now-1m.." },
  "goodDirection": "down"
}
```

```
+----------------------------+        +----------------------------+
|  Churned this month        |        |  Storage used              |
|                            |        |                            |
|        24                  |        |        95 GB               |
|  [ +10% ]  <- red          |        |  [==================· ]    |
+----------------------------+        |    ^ red fill, red track   |
   up is BAD, so a rise is             +----------------------------+
   bad news — but "+10%" still            "goodDirection": "down"
   says which way it moved.               + display/target → 95% of
                                          quota ramps to danger.
```

`goodDirection` must have something to drive. Declared on a plain scalar with
**no `compare`** (no delta badge) and **no `display` + `target`** (no meter), it
would decode and change nothing on screen — so declare **rejects** it with
`invalid_metric_good_direction`. Add one of the two signals, or leave the field
off. Omitting it is always valid, and is the honest choice for a metric with no
polarity (a headcount).

### `trend` — a time-series sparkline

`trend` draws the metric's **own measure over time**, as a small line under the
figure:

| field | | |
|---|---|---|
| `property` | optional | the **timestamp** property to bucket by. Default: `created_at`. |
| `bucket` | required | `"day"` \| `"week"` \| `"month"` \| `"quarter"` \| `"year"` — one bucket is one point. |
| `periods` | required | how many points to plot: an integer **2–24**. |

```json
{
  "type": "metric",
  "source": {
    "objectType": "invoice",
    "aggregate": { "op": "sum", "property": "amount" }
  },
  "label": "Revenue",
  "format": "currency",
  "compare": { "created_at": "$now-1m.." },
  "goodDirection": "up",
  "trend": { "bucket": "month", "periods": 12 }
}
```

```
+----------------------------+
|  Revenue                   |
|                            |
|        € 1.2M              |   <- the figure
|  [ +8% ]                   |   <- the delta badge (from `compare`)
|            .        ,-*    |   <- the trend: 12 monthly points,
|   .-.  ,--'  `-.  ,-'      |      grey line, the CURRENT period
|  '   `'         `'         |      marked in the accent colour
+----------------------------+
```

**The metric is still a scalar.** `trend` is decoration *about* the number, not
a second data shape — the figure, the delta, and any meter all still read one
value. This is why `source.groupBy` remains **rejected** on a metric
(`invalid_metric_group_by`): a grouped source would leave the figure nothing to
show. Declare the series here instead. For a full chart with axes, a tooltip,
and a longer series, use a [`chart` block](#7-chart--a-bar-or-line-chart) with
`groupBy: "created_at@month"` — that is also the answer when you want more than
24 points.

`trend` and `compare` are **independent**: `compare` drives the delta badge (one
prior-window read), `trend` drives the sparkline (one bucketed read). Declare
either, both, or neither. The sparkline composes with every `display` variant,
including `"hero"` and the meters.

Rules worth knowing:

- **`periods` is a cap, not a promise.** A metric with five months of history
  plots five points, not seven zeros and five values — absent data is *unknown*,
  not zero. Gaps *inside* the range are real zeros and are drawn.
- **The bucketed property must be a `timestamp`.** Declare rejects anything else
  with `invalid_metric_trend` — including the `created_at` default when the
  ObjectType doesn't have one, in which case name a `property` explicitly.
- **Not on a record-scoped source.** `{ link, aggregate }` has no bucketed read,
  so a `trend` there would draw nothing; declare rejects it
  (`invalid_metric_trend`). Use `{ objectType, aggregate }`.
- The sparkline has no axes and no tooltip, and is hidden from screen readers —
  it is decoration. The figure and the delta badge carry the meaning.

---

## 7. `chart` — a bar or line chart

The same `AggregateSource`, **drawn**: `groupBy` is the dimension axis,
`aggregate` the measure axis. `groupBy` is REQUIRED (a source without one is a
scalar — use `metric`; declare rejects it with `invalid_chart_group_by`).

`kind` (`"bar"` | `"line"`) is **optional and inferred**: a categorical
`groupBy` → `bar`, a time-bucketed `groupBy` (`created_at@month`) → `line`. Set
it only to override. `format` hints the VALUE axis; with `"currency"` the code
comes from the aggregated property's own declared currency (never from this
spec). ONE measure always — two measures is two charts, never a second y-axis —
but `seriesBy` splits that measure into series (grouped/stacked bars,
multi-line) with a legend; without it, one series, one hue, no legend. A table
view is always one click away. Full rules:
[viewspec-analytics](viewspec-analytics.md).

```json
{
  "type": "chart",
  "source": {
    "objectType": "deal",
    "aggregate": { "op": "sum", "property": "amount" },
    "groupBy": "created_at@month"
  },
  "format": "currency",
  "title": "Revenue by month"
}
```

```
+--------------------------------------------+
|  Revenue by month           [ Show table ] |
|   €600k |            .-*                   |
|         |        .-*'                      |
|   €300k |    .-*'                          |
|         | .-*                              |
|      €0 +------------------------------    |
|          Jan   Feb   Mar   Apr   May       |
+--------------------------------------------+
```

---

## 8. `file` — image or download

Renders an image (for image content-types) or a download link. The `source`
is one of two shapes: linked attachments (`{ objectType }`) or a single
`file`-typed property (`{ property }`).

```json
{
  "type": "file",
  "source": { "property": "logo" }
}
```

Attachments form — every file attached to the row in scope (0..N):

```json
{
  "type": "file",
  "source": { "objectType": "customer" }
}
```

Attachments are linked to a **row**, not to a named collection: this renders
all files attached to the customer in scope, newest first. There is no way to
split them into named groups (a `"ref"` field is rejected at declare — it never
did anything). To surface one specific file on its own, give the ObjectType a
`file`-typed property and use the `{ "property": … }` form above.

```
+--------------------------------------+
|  Logo                                |
|   +--------+                         |
|   | [IMG]  |                         |
|   +--------+                         |
|                                      |
|  Documents                           |
|   • contract.pdf      [ download ]   |
|   • invoice-01.pdf    [ download ]   |
+--------------------------------------+
```

---

## 9. `board` — a kanban board

Groups an `ObjectTypeSource` into columns by a required `groupBy` enum
property. Per-card fields use the same `ColumnRef` grammar as `table`.
`columns` is optional — omit it to fall back to the ObjectType's primary
properties. `rowActions` (an array of ActionType names) optionally surfaces
per-card action triggers. `title` is optional section-chrome rendered above
the board. `source` must be an `ObjectTypeSource` (not aggregate-backed).
`createAction` (optional) names an ActionType — typically `<type>.create` — that
adds a board-level '+ New' control and a per-column '+ new' in each column
header; the per-column control presets the new record's `groupBy` value to that
column. Absent → no create control. Validated at declare time
(`unknown_action`).

```json
{
  "type": "board",
  "source": { "objectType": "deal" },
  "groupBy": "stage"
}
```

```
+------------+  +----------------+  +--------+  +--------+
|  open      |  | proposal_sent  |  | won    |  | lost   |
+------------+  +----------------+  +--------+  +--------+
| Acme Corp  |  | Beta Ltd       |  | Gamma  |  |        |
| € 12,000   |  | € 45,000       |  | € 8k   |  |        |
+------------+  +----------------+  +--------+  +--------+
```

With per-card fields, a title, and row actions:

```json
{
  "type": "board",
  "source": { "objectType": "deal" },
  "groupBy": "stage",
  "title": "Deal Pipeline",
  "columns": ["name", "amount"],
  "rowActions": ["deal.advance"],
  "createAction": "deal.create"
}
```

---

## 10. `queue` — a prioritized action worklist

A `queue` reframes a view as a **worklist of jobs**, not a table of records.
The operator lands on it and clears it top-down — each row carries only the
decision-context needed to act (a title, a subtitle, a few facts, an urgency
signal, an age) and the actions are **inline on the row**. It is keyboard-first
triage with an act-in-place row-exit, a one-at-a-time focus mode, and inbox-zero
as the reward. It is **additive** to the palette: the default landing for
action-oriented apps, sitting *alongside* `table`/`board`/`detail` — not a
replacement. Browse, search, reporting, and bulk edits stay on tables.

Both `title` and `why` are **required** — `why` is the one-line "why now" that
justifies the whole queue (e.g. *"Flagged at_risk — oldest first."*). `source`
is the same `ObjectTypeSource` `table`/`board`/`detail` use — but a queue is
ordered **only** through `order`, so setting `source.sort` on a queue is
**rejected at decode** (clear error pointing you at `order`). `order` is the sort
(**urgency IS the sort** — no separate priority field); the first `order` key
also drives the aging badge on each row. `urgency` (optional) names an enum
property whose declared `valueColors` tint the **whole card border** in the
row's urgency color. `buckets` (optional) names a **date/timestamp** property
(`{ "from": "due_at" }`) and groups rows into a **fixed, renderer-owned scheme —
Overdue / Today / This week / Later** — computed from that field vs. now. The
scheme is closed: you name only the field, never the bucket labels or their
order. Absent ⇒ a flat, ungrouped queue. It is orthogonal to `urgency` (border
tint) and `order` (sort). `row` is the
scannable job-line: `title` (required), optional `subtitle`, up to three `facts`
(each a template string, or `{ value, label?, format? }` where `format` ∈
`text` | `number` | `currency` | `percent`), and an optional `peek` naming a
`detail` view to inline-expand in the card.

**Referencing record fields in `row`.** `title`, `subtitle`, and a fact's
`value` resolve field references against each record. Two forms:

- A **bare property name** — `"name"` — resolves to that record field's value,
  the SAME "property name → value" grammar `table` columns and `detail` fields
  use. This is the form to reach for. (A string that is *not* an actual property
  on the record passes through as a literal label, so `title: "Overdue"` still
  reads as text.)
- A **`$field` template** — `"$name"`, or a composite like
  `"$owner · {{age at_risk_since}}"` — for interpolating one or more fields into
  a single string, plus the `{{age <field>}}` helper (a compact relative age like
  `12d`/`now` from a timestamp). A timestamp-typed field is humanized to a
  friendly date; a missing field resolves to empty and dangling `·` separators
  are trimmed.

So `title: "name"` and `title: "$name"` both render the record's `name`; use the
`$…`/`{{…}}` template form only when you need to combine fields or use a helper. `actions.primary` is **required**
(the action a row is cleared with); `secondary[]` and `defer` are optional.
`empty` is the inbox-zero state. `page_size` (optional, integer 1–200;
default 50) sizes the queue's first page and each auto-loaded page.

```json
{
  "type": "queue",
  "title": "At-risk",
  "why": "Flagged at_risk — oldest first. Log a touch or snooze to clear.",
  "source": {
    "objectType": "customer",
    "filter": { "status": "at_risk" }
  },
  "order": [{ "field": "at_risk_since", "dir": "asc" }],
  "urgency": { "from": "status" },
  "row": {
    "title": "$name",
    "subtitle": "$owner · {{age at_risk_since}}",
    "facts": [
      { "value": "$nps", "label": "NPS", "format": "number" },
      { "value": "$mrr", "label": "MRR", "format": "currency" }
    ],
    "peek": "Customer.detail"
  },
  "actions": {
    "primary": { "action": "customer.touch", "label": "Log touch", "key": "Enter" },
    "secondary": [{ "action": "deal.open", "label": "Open deal" }],
    "defer": { "action": "customer.snooze", "label": "Snooze", "key": "e" }
  },
  "empty": { "art": "inbox-zero", "title": "Queue clear", "body": "All caught up." }
}
```

```
  At-risk
  Flagged at_risk — oldest first. Log a touch or snooze to clear.   [ Filter ] [ Focus F ] [ ? ]
+-------------------------------------------------------------------------------+
|  NW  Northwind                     NPS 5   MRR 4.200 kr   12d   [Log touch ▸][Snooze][Open deal]  |
|      Ada · 12d                                                    ⌄            |
+-------------------------------------------------------------------------------+
|  CG  Contoso                       NPS 6   MRR 9.100 kr    3d   [Log touch ▸][Snooze][Open deal]  |
|      Ben · 3d                                                     ⌄            |
+-------------------------------------------------------------------------------+
```

**How it renders (recent owner decisions — document these accurately):**

- **All action buttons are always visible** on every row. There is no
  hover/selection progressive disclosure.
- **No urgency color rail.** Urgency reads from the **sort order** plus the
  right-aligned **aging badge** (which turns urgent past ~14 days). When
  `urgency` names an enum with declared `valueColors`, the resolved token tints
  the **whole card border** in that urgency color (a non-neutral `hot|warm|cool|
  info`; `neutral`/absent keeps the default border) — there is no separate left
  rail or status-chip tint.
- **Date buckets** (`buckets.from`) group the ordered rows under **Overdue /
  Today / This week / Later** section headers; without `buckets` the queue is a
  single flat list.
- **Peek unfolds inside the card** — an inline accordion with a rotating caret,
  rendering the referenced `detail` view in place (not a detached panel).
- **Keyboard triage:** `j`/`k` move · `Enter` primary · `1`–`9` Nth secondary ·
  `e` defer · `Space` peek · `/` filter · `?` shortcuts · `F` focus mode.
- **Focus mode (`F`)** is a one-at-a-time full-screen overlay with a progress
  bar and inbox-zero when the queue clears.
- **Act-in-place:** invoking an action optimistically flies the row out and
  advances selection, with an undo toast; a transport failure rolls it back.
  In the worked example both actions move the customer off the `at_risk` filter,
  so the row leaves the queue the moment you handle it.

**It raises the data-model bar.** A queue is only as good as its "why now"
signal, so the ObjectType must model **status enums** (with `valueColors` for
the chip) and **aging timestamps** (the `order`/`{{age}}` field). Without them
the queue confidently shows the wrong thing.

**Authoring gotchas (learned building the worked example):**

- `why` is **required** — the schema rejects a queue without it.
- Each row action derives its subject from the target ObjectType via the
  `<object_type>_id` input-param convention (e.g. `customer.touch` takes a
  `customer_id` input). A **mismatched param name = "no subject" = dead row
  buttons** — `catalog.apply` now fails this at apply time
  (`unresolvable_subject`).
- The bound actions need `governance` and `idempotency` set (strict apply
  validation, ADR-0019).
- `row.peek` names a declared `detail` view; an unresolvable ref fails apply
  (`unresolvable_ref`).

A complete, applied-and-live-verified worked bundle — object type with
`valueColors`, two act-in-place actions, the queue + a plain table + the peek
`detail`, and an app that navs them — lives at
[`examples/queue-demo/canon/`](../../../../examples/queue-demo/canon). Its
[README](../../../../examples/queue-demo/README.md) walks the apply + seed loop.

---

If a screen requires UI that no palette block covers, that is a
Hosted/External-tier concern. The Tier-1 palette grows first-class variants
on pull.
