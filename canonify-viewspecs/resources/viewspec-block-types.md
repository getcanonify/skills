# The 10 block types

One reference per block: its fields, one example, and what the renderer
decides for you (none of those are knobs). Every block also takes `showWhen`
(`[{ field, op: "eq", value? }]`, all must hold — UX only) and, except `queue`
where it is required, `title`, painted once in the block header. Sources,
filters, columns and the aggregate envelope are in
[viewspec-binding-and-sources.md](viewspec-binding-and-sources.md).

**Enum colours and labels.** An enum with no declared `valueColors` renders
with a neutral tint everywhere it appears — colour is never inferred from the
value. To give a status, tier or channel semantic colours,
declare `valueColors` on the ObjectType (the token table is in the
`canonify-objects-actions` skill's
[object-property-types.md](../../canonify-objects-actions/resources/object-property-types.md));
that one mapping colours chips, board columns, chart bars and queue rails.
`valueLabels`, declared beside it, supply the words; an undeclared value shows
its humanized raw value. The stored value stays the filter and sort key.

## 1. `table` — a list

Required `source`, `columns` (≥ 1). Optional `rowActions`, `presets`,
`page_size`, `rowGroups`, `totals`, `actions`, `compare`, `window`. With an
aggregate `source` it is a grouped summary (below).

```json
{
  "type": "table",
  "source": { "objectType": "ticket", "sort": "-created_at" },
  "columns": ["subject", { "ref": "priority", "sortable": true }, "status", "comments.count"],
  "presets": [
    { "label": "Open", "filter": { "status": { "in": ["open", "pending"] } } },
    { "label": "This week", "filter": { "created_at": ":this_week" } }
  ],
  "rowActions": [
    { "action": "ticket.resolve", "label": "Resolve", "default": true },
    { "action": "ticket.escalate", "presetParams": { "reason": "$row.status" } }
  ]
}
```

- **`presets`** — declared one-tap chips `{ label, filter }` (non-empty
  label; the filter grammar and closed paths of `source.filter`). They are
  alternative views, not stacking filters: choosing one clears the keys only
  the others set; tapping the active one clears it. A chip writes over
  `source.filter` per key (a key both name is replaced) and ANDs across keys.
  Each chip shows the live count its tap would show. With no chip active the
  list is the unfiltered `source`; an explicit "All" is legal as `{ "label":
  "All", "filter": {} }` — it counts the whole source and a tap clears the
  other presets, but it never shows as the active chip.
- **`rowActions`** — replaces the inferred row actions with a curated list;
  the header keeps the type's create, and an action curated onto the rows is
  not repeated in the header. Each entry is an action name or `{ action,
  label?, default?, showWhen?, presetParams?, sources? }` (`board` takes the
  same grammar). `$row.<field>` and `showWhen` fields are checked against the
  source type; an unknown action is `unknown_action`. A transition of the
  row's type shows only on rows in one of its `from` states, with no
  `showWhen`. This `showWhen` is the view's own, `eq`-only filter; the
  action's `presentation.showWhen`
  ([actions-input-governance.md](../../canonify-objects-actions/resources/actions-input-governance.md#presentation))
  is its precondition on every surface (`eq|neq|in|present`) — both must
  hold. An entry's caption is its `label`, else the view's `actions`, else
  the owning type's `display.actions`, else `presentation.label`. Board
  cards and queue rows apply the same gates and caption order (a queue has no
  view step). The first `default: true` action renders inline;
  of the rest a lone survivor renders inline and two or more go behind `⋯`
  (counted per row after `showWhen` and permissions). Without `rowActions`,
  inferred actions drop any action its **owning** ObjectType hides in
  `display.actions`.
- **One tap or a dialog.** An action whose every input is preset by the row
  (subject included) fires on one tap unless it declares
  `presentation.confirm`, is destructive or needs approval; otherwise it opens
  a dialog titled `<Action> · <record>`.
- **`totals`** — names shown `number`/`money` columns of the source type to sum
  in a "Total" footer, formatted like the cells; it sums the whole filtered
  list (a server `sum` while pages remain). Anything else is
  `invalid_table_totals`; on an aggregate table `totals_on_aggregate`.
- **On a record page** a record-scoped table renders like the same table at
  top level: its `columns`, `totals`, sort, `presets`, `rowGroups` and
  `rowActions` are kept, in one card under the section heading. Its header
  offers an action about the parent that inserts the child (`job.add_part` →
  `job_line`, discovery `creates`; the record bound) in place of the child's
  generated create, else the child's creates with the parent FK bound; an
  action offered there is not repeated in the record's toolbar.
- **`rowGroups`** — `{ by, groups?, defaultCollapsed? }` sections the rows
  under collapsible label + count headers; every row is kept. `groups` gives
  known values a label and order (declared groups render even when empty,
  others follow, raw-labelled); sections start expanded unless
  `defaultCollapsed: true`. `by` must exist (`invalid_row_groups_by`); on an
  aggregate table it is `row_groups_on_aggregate`.
- **`actions`** (table, board) — `{ "<action>": { label?, emphasis? } }`
  relabels an action or promotes it (`emphasis: "primary"`) in this view only.
  Naming an action also offers it in a top-level table's header (an action
  whose subject is the row type stays on the rows); a `primary` one gets its
  own header control instead of sharing `⋯`.

| Grouping knob | Groups | Effect |
|---|---|---|
| `rowGroups.by` | a table's rows | collapsible sections, every row kept |
| aggregate `groupBy` | data | one measure row per group |
| `board.groupBy` | cards | one column per enum value |
| `queue.buckets` | queue rows | fixed Overdue / Today / This week / Later |

**The chrome scales with the content.** A list whose unfiltered total is
≤ 10 rows drops search and `+ Filter` (declared presets stay). A table in a
`grid` cell beside other blocks is a summary widget: no search, `+ Filter`,
inferred create or inferred row menu (declared `presets`, `rowActions` and
`actions` entries stay). A table whose `source.filter` pins a column to the
viewer (`$principal` / `$actor`: an inbox, "my tasks") offers no inferred
header actions; name one in `actions` to offer it. The title cell leads with a
glyph only when it carries information — the row's own image (an `https` URL in `avatar_url`, `photo_url`, `image_url`,
`logo_url`, `avatar`, `photo`, `image` or `logo`) or initials for a person-like
type. A table fits its card (measured on the card, not the screen): while
columns overflow, the lowest-value one folds into the title cell's second line
— plain text first, then dates and FK names, other figures and chips, the
status chip, the primary money last; ties fold the later column first. The
order is derived, not a knob. Only when the title column alone cannot fit does
the table scroll inside its card, with the `↔` cue while it overflows.

**Grouped summary table.** With an aggregate `source`, each column is a
dimension (the `groupBy` path without its `@bucket`, or the `seriesBy` path), a
measure (`count`, `sum:<p>`, `avg:<p>`, `max:<p>`), or the source measure's
bare property; anything else is `invalid_aggregate_table_column`. Measure
columns take `label`/`format` (default "Count", "Sum of Amount") and render in
the order listed; with `seriesBy` or `compare` the table carries only the
source measure. Enum dimensions follow the declared value order and render
as their chips.

```json
{
  "type": "table",
  "title": "Work by bike type",
  "source": { "objectType": "job", "aggregate": { "op": "count" }, "groupBy": "bike.kind" },
  "columns": ["bike.kind", { "ref": "count", "label": "Jobs" }, { "ref": "sum:quote", "label": "Quoted" }]
}
```

## 2. `detail` — the record surface

Required `source`; optional `recordLayout`, `properties`, `header`, `links`.
A standalone `detail` is the default master-detail record surface: the platform
derives the list rail from its `source` (`objectType`, `filter`, `sort`;
with no `sort` it opens newest first) and shows the selected record in one
lifted panel — no second `table` needed. The
record id comes from `$context.<objecttype>.id`, then `$context.id`.

```json
{
  "type": "detail",
  "source": { "objectType": "customer", "sort": "name" },
  "recordLayout": "master-detail",
  "header": { "title": "name", "subtitle": "{lifecycle_state} · CVR {organization.cvr}", "status": "tier", "metrics": ["lifetime_value", { "ref": "balance_due", "label": "Due" }] },
  "properties": ["email", { "ref": "created_at", "label": "Customer since", "format": "date" }],
  "links": [
    { "link": "appointments", "as": "tab" },
    { "link": "orders", "as": "card", "limit": 3, "sort": "-created_at" },
    { "link": "notes", "as": "chip", "label": "Notes" }
  ]
}
```

- **`properties`** — ordered refs or `{ ref, label?, format? }`; omitted, the
  grid shows `display.primaryProperties` above the fold and the rest under
  "More details". The body never repeats what the header shows, and an unset
  field prints no row.
- **`header`** — `title`, `subtitle`, `status` (an enum), `metrics`; each slot
  optional, each a ref or link traversal (`organization.cvr`) resolved against
  the linked record; an FK ref (`customer_id`) shows the linked record's
  title. A `subtitle` may be a template of `{ref}` placeholders: an
  enum placeholder shows its value label, a null one an em-dash in place, an
  unknown ref is `unknown_column_ref`. `title` and `status` take one ref, never
  a template (`"{code} {title}"` is `unknown_column_ref`); omit `title` and the
  heading is the record's `display.title`, which may be a `{ template }`
  ([object-property-types.md](../../canonify-objects-actions/resources/object-property-types.md)).
  `metrics` takes column refs; omitted, a
  currency-declared money field becomes a header metric on its own.
- **`recordLayout`** — omitted or `"master-detail"` is the record surface.
  `recordLayout: "flat"` is a spatial opt-out only: a flat detail does not opt out of the links contract.
  `links[].as` still presents related records; only the rail, the lifted
  panel and the mobile shell go.
- **`links`** — `{ link, as, limit?, sort?, label? }` over the type's
  `cardinality: "many"` links. `as` is `tab` (a list the reader works in),
  `card` (a glanceable preview, `limit` rows, default 3) or `chip` (a count in
  the header); `sort` orders the linked rows, `label` renames. These shape
  presentation only, never what is readable. There is no implicit `Overview`
  tab: the header, grid, cards and chips stay on screen and full lists hang
  below — two or fewer stacked as sections, three or more in tabs landing on
  the first non-empty one. Do not add an `Overview` entry.
- **Declared wins, empty wins.** A declared link with rows keeps its declared
  `as`; a link whose live count is 0 renders as one dimmed `0` chip whatever
  you declared. Undeclared links use the fallback: ranked by live count, the
  two highest-count non-empty links become full lists and the rest chips —
  so an undeclared record's layout shifts as data grows. Declare `links` on
  any record with several many-links.
- **Mobile.** From 900px wide the rail (260px; 480px from 1200px when an
  explicit rail table declares more than two columns) sits beside the panel.
  Below 900px it is a two-state drill-down: the list state (full-width list,
  search, filters; a no-id entry lands here) and the record state (a deep link
  with an id lands here, even off the loaded page). **Back to list** is a
  state transition back to the list state that keeps the selection and
  restores focus; previous/next and keyboard navigation stay available.

**Where a row click lands.** A row opens the auto record page unless a
declared view can render one record of that type (a `detail`, or a
`composite` with a `detail` child; never a view another view uses as a queue
`peek`). Then the row opens that view on the record route. With an app active
this applies only to types the app's nav surfaces (a routing rule, not a
data boundary); among candidates a view in
the app's nav wins, then the ascending view name. On that route a composite is
one record page: one action toolbar (non-owner details hide theirs), child
tables auto-scoped to the record, header refs traversing links.

**An explicit composite** of a `table` and a `detail` is still accepted when a
page needs bespoke rail columns or curated `rowActions`. A rail row is
navigation: it offers no inferred actions (the selected record's toolbar
does). Its folded columns make one row of two lines: the title, then ONE
secondary line — values that name themselves (text, linked titles, chips)
before labelled ones (figures, dates), cut with an ellipsis and whole in its
tooltip — with the status chip at its end; the primary money (the total) stays
in the record pane. A real `table` keeps the full fold described above.

## 3. `panel` — an action form

`actionForm: { action, submitLabel?, presetParams?, sources? }`. The form is
built from the action's input schema and its `presentation` (field order
`formFields`, opening values `inputDefaults`, picker scopes `inputFilters` —
see the `canonify-objects-actions` skill's
[actions-input-governance.md](../../canonify-objects-actions/resources/actions-input-governance.md)),
so every surface that opens the action shows the same form. `title` and
`submitLabel` default to `presentation.label`; without one, no heading and the
localized Submit. A preset input is submitted but not shown — to open the form
on an editable value use `inputDefaults`. A date-time input edits date and time
on the org's clock and submits the UTC instant; a `money` input shows its
currency and takes major units. Submission goes through the governed invoke.

```json
{ "type": "panel", "actionForm": { "action": "customer.create", "submitLabel": "Add customer", "presetParams": { "tier": "standard" } } }
```

**Subject auto-bind.** An action's subject ([the
rule](../../canonify-objects-actions/resources/actions-input-governance.md#subject))
is filled from the record in scope for a panel, button, row, card or toolbar —
never write `presetParams: { "deal_id": "$context.id" }`.
Precedence: explicit `presetParams` > auto-bind > the form asks. A top-level
list has no record, so `<type>.create` asks for its inputs. `sources: {
"<input>": { action, read } }` seeds an input from another action's result.

## 4. `button` — a one-tap action

`action`, optional `label` (a literal or `@t:` key; default the action's
`presentation.label`, then a localized CRUD verb + type name), `presetParams`,
`sources`. Emphasis, hiding and the caption across all surfaces belong on the
ObjectType's `display.actions`, not on each button.

```json
{ "type": "button", "action": "deal.advance", "label": "Advance stage" }
```

## 5. `composite` — a container

`children` (non-empty, full ViewSpecs, nesting to any depth), optional
`layout`, `filters`.

- **`layout`** — `stack` (default), `grid` or `tabs`; no CSS knobs. In a grid a
  child may be `{ span: 2 | 4, view }` (`invalid_composite_span` outside a
  grid); the grid is one column on a phone, two on a tablet, four wide — but a
  grid of only single-span `metric`s stays two-up on a phone. In `tabs` each
  child's `title` is its tab label and is required (`invalid_tabs_child`); a
  child hidden by `showWhen` loses its tab too.
- **Child types.** A child may bind a different existing ObjectType than the
  view's `object_type` (an `invoice` detail over an `invoice_line` table); a
  child naming a type that does not exist is `source_object_type_mismatch`.
  In a record-scoped composite, a child `table` with no filter is auto-scoped
  through the one many-link from the parent to its type; with two such links
  nothing is scoped (add a filter); with none the table stays org-wide —
  declare the link or a filter. A filter on the FK means you own the scoping.
- **`filters`** — one reader filter row `[{ property, kind, required? }]` that
  overlays (replaces) that property on every child that carries it, so the
  figures on the page agree; other filters stay. `kind`: `date-range` (a
  timestamp; All time / Today / Last 7 days / Last 30 days — longer and custom
  ranges are not offered), `dimension` (an enum), `reference` and `date`.
  `reference` filters render a reader picker for a to-one foreign-key path;
  `date` filters render a date picker for a date/timestamp path. Paths use the
  closed filter grammar per child; a property no direct child can honour, or a
  wrong type, is `invalid_composite_filter`.
  Set `required: true` when a record-scoped composite must not render until its
  `$context.*` reference is available. It holds the child at "Pick a …"; it is
  a rendering contract, not security.
  Record-scoped composites use `$context.<property>` in child source filters,
  for example `{ "order.location_id": "$context.location_id" }`.
- **One heading, one declaration.** Two identical sibling blocks (same kind
  under one title, or identical apart from `title`/`label`) are
  `duplicate_block_declaration`; two blocks that paint the same heading are
  `duplicate_block_heading`. An untitled table paints its type's plural, a
  grouped table or chart "<Types> by <Dimension>", a titleless panel its action
  caption, a detail the record name; an untitled composite's children count as
  siblings of its neighbours. Differently titled sections, separate tabs and
  mutually exclusive `showWhen`s do not collide; headings compare in every
  shipped locale. Give one block a distinct `title`.

## 6. `metric` — a KPI tile

`source` (aggregate, or record-scoped `{ link, aggregate }`), `label`
(required), optional `format`, `compare`, `window`. Without `groupBy` it shows
one figure. With a `@bucket` spine it reads one bucketed aggregate: the figure
is the newest bucket (the one containing now when the measure zero-fills), `window` draws a sparkline of the newest N (current
period in the accent), and `compare` makes a delta badge — `offset` against the
same bucket a period ago (wins when both are declared), `rolling` against the
trailing mean. The badge names its reference ("vs Sep", "vs Oct 2025"); a
running bucket is held against the same elapsed days ("vs Sep 1 – 16"); a null or
zero reference shows an em-dash; a rolling-only tile on a partial bucket shows
no delta and "so far". A categorical `groupBy` or a `seriesBy` is
`invalid_metric_group_by`.

```json
{
  "type": "metric",
  "source": { "objectType": "order", "aggregate": { "op": "sum", "property": "total" }, "groupBy": "created_at@month" },
  "label": "Revenue",
  "window": { "last": 12 },
  "compare": { "offset": { "period": "year", "periods": 1 } }
}
```

A record-scoped metric aggregates the current record's rows over ONE link:
`{ "link": "orders", "aggregate": { "op": "sum", "property": "total" } }`. A
dotted `customer.orders` is refused; the property must exist (numeric for
sum/avg) on the link target.

## 7. `chart` — bar or line

`source` with `groupBy` (else `invalid_chart_group_by` — use a metric);
optional `kind`, `stacked`, `format`, `compare`, `window`, `partialTrailing`.
One measure always (two measures is two charts); `seriesBy` splits it.

```json
{
  "type": "chart",
  "source": { "objectType": "job", "aggregate": { "op": "count" }, "groupBy": "promised_at@month", "seriesBy": "bike.kind" },
  "title": "Jobs by month, by bike type"
}
```

- **`kind`** — inferred: a categorical `groupBy` draws `bar`, a bucketed one
  `line`; set it only to override. An unknown kind fails at decode.
- **`stacked: true`** — segments that compose a total; needs `seriesBy`, `bar`
  (set `kind: "bar"` on a time spine) and `count`/`sum`
  (`invalid_chart_stacked`). Without it series are grouped side by side.
- **`format`** — the value axis and tooltip; `currency` takes its code from the
  property, and a property with no currency renders a bare number.
- **`partialTrailing: false`** — opts out of marking the bucket containing now
  (hatched bar, or a dashed final line segment, plus a caption). On a
  categorical spine it is `invalid_chart_partial_trailing`. The read's
  `partial: true` is unaffected; a standalone grouped table never marks it.
- **`compare`** — the prior layer draws dashed in the base hue, the rolling
  mean thin and muted, and the legend turns on.

What the renderer decides: enum dimensions keep their declared order and wear
their `valueColors` (series wear them only when every series gets a distinct
one, else `--chart-1..8`); other categories sort by value; time is
chronological. Beyond 12 categories (8 series) the tail folds into **Other** —
summed for count/sum, max-composed for max; an `avg` tail is dropped and the
caption says so. Long labels flip bars horizontal. Series take a fixed hue
order by identity, never magnitude; a legend names every series in text (end
labels with ≤ 4 lines); the tooltip lists all series at a point. A table view
is one click away, with the same split, compare columns and window. A capped
read always says so in the caption.

## 8. `file` — image or download

`source` is `{ property }` (one `file` property; `invalid_file_source` if it
is missing or not a file) or `{ objectType }` — every file attached to the
record in scope, newest first. Images render inline, others as downloads.

```json
{ "type": "file", "source": { "property": "logo" } }
```

## 9. `board` — a kanban

Required `source` (records) and `groupBy` (an enum); optional `columns` (card
fields; default the primary properties), `rowActions` (the table grammar),
`createAction`, `overdue`, `lanes`, `actions`.

```json
{
  "type": "board",
  "source": { "objectType": "job", "sort": "promised_at" },
  "groupBy": "status",
  "columns": ["customer_id", "bike_id", "quote"],
  "overdue": "is_overdue",
  "lanes": { "collected": { "sort": "-collected_at", "limit": 5 } },
  "createAction": "job.check_in"
}
```

- **Columns the data can fill.** An equality or `in` filter on the `groupBy`
  property limits the columns to the values it admits. Columns share the
  width down to 10rem each once the board is ≥ 48rem wide (16rem on a
  narrower board), then scroll inside the board.
- **`lanes`** — `{ "<value>": { sort?, limit?, create? } }`; a lane with `sort`
  or `limit` is read on its own (count badge = the column's total, "Showing N
  of M", Load more). A key that is not a declared value is
  `invalid_board_lane`.
- **'+ New' only where it lands.** `createAction` adds a board-level '+ New';
  a column gets its own from `lanes.<value>.create`, else `createAction`, only
  where that create puts the card. Under a StateMachine on `groupBy`, a plain
  create lands in the machine's `initial` state column and a
  create-then-transition action in its marker's `to` column (see the
  `canonify-app-authoring` skill's
  [patterns.md](../../canonify-app-authoring/resources/patterns.md)); an
  ungoverned enum lands in any value its `groupBy` input admits (the column
  presets it). A lane `create` that cannot land is
  `invalid_board_lane_create` — `catalog validate`, `catalog apply`,
  `view_specs.declare` and `view_specs.preview` run the same rule.
- **One labelled move per card.** Of the live actions (a transition is live in
  its `from` states; `showWhen` applied) the primary is the `default: true`
  one, else `emphasis: "primary"`, else the first; one other renders inline,
  more go behind `⋯`.
- **`overdue`** — a boolean property (often a derived `is_overdue`;
  `invalid_board_overdue` otherwise): a passed date paints late exactly where it
  is true. Without it a passed date paints late on a card whose state still
  has a move.
- **Card hierarchy** — the heading is the record's `display.title` (a
  column the title already says — its property, `via` or a template token — is
  not repeated; only a type with no human title leads with its first column),
  then one muted `·` line of values, a footer of dates and enum/boolean chips;
  an FK column reads as the linked record's title text, so name the FK
  (`bike_id`), not a link path. Empty values, `false` flags and a `neutral`
  enum value among coloured ones paint nothing.

## 10. `queue` — a worklist cleared row by row

Required `title`, `why` (the one-line "why now"), `source` (records, no
`sort`), `order`, `row`, `actions`; optional `urgency`, `buckets`, `empty`,
`page_size`. The landing for action-oriented work, beside tables and boards.

- **`order`** `[{ field, dir }]` is the urgency; a timestamp first key drives
  the age badge, which names its field and reads in the viewer's locale
  ("Promised in 3 days").
- **`urgency: { from }`** — an enum whose `valueColors` paint a rail down the
  card's leading edge (`neutral` paints none) and a chip among the facts.
  **`buckets: { from }`** — a date field
  grouping rows under Overdue / Today / This week / Later (fixed labels).
- **`row`** — `title`, `subtitle?`, `facts?` (≤ 3 `{ value, label?, format? }`,
  `format` ∈ `number` · `currency` · `percent`; a fourth is
  `invalid_queue_row`), `peek?` (a declared `detail` view, unfolding in the
  card; unknown is `unresolvable_ref`). The peek shows the record's
  properties (the view's `properties`, no second header, metrics or linked
  lists) and its OTHER actions — never one its row already offers, and no
  generated create/update/delete (those are the record page's). A text is a property name (else a
  literal) or a `$field` template with `{{age <field>}}`; timestamps humanize,
  missing fields drop with their `·`. An FK (`$customer_id`, a `mechanic_id`
  fact) renders as the linked record's title. A money fact renders in its
  currency; an enum fact is its chip, and a `neutral` value among coloured
  ones paints none (the board card's rule).
- **`actions`** — `primary`, `secondary?`, `defer?`, each a name or `{ action,
  label?, key?, showWhen? }`. Per row, an action survives only if its
  `showWhen` holds and, for a transition, the row is in a `from` state; the
  row's primary is the first survivor of `[primary, ...secondary]` and answers
  to the primary's `key`; failing both, a surviving `defer` stands in. One
  queue can therefore hold rows in several states. Each action needs an input
  carrying the row id (`unresolvable_subject`).

```json
{
  "type": "queue",
  "title": "Workbench",
  "why": "Every open job, soonest promise first.",
  "source": { "objectType": "job", "filter": { "status": { "in": ["booked", "in_progress", "ready"] } } },
  "order": [{ "field": "promised_at", "dir": "asc" }],
  "buckets": { "from": "promised_at" },
  "row": { "title": "title", "subtitle": "$customer_id · {{age created_at}}", "facts": [{ "value": "quote", "label": "Quote" }] },
  "actions": {
    "primary": { "action": "job.start", "label": "Start", "key": "Enter" },
    "secondary": [
      { "action": "job.mark_ready", "label": "Mark ready" },
      { "action": "job.call_customer", "label": "Call", "showWhen": [{ "field": "priority", "op": "eq", "value": "rush" }] }
    ]
  },
  "empty": { "title": "Bench clear", "body": "All caught up." }
}
```

How it behaves: every row shows all its buttons (key hints on the selected row
only); keys `j`/`k` move, `Enter`
primary, `1`–`9` the Nth secondary, `e` defer, `Space` peek, `/` filter, `?`
help, `F` a one-at-a-time focus mode ending in inbox-zero. Keys act on the
selected row while it (or nothing) holds focus; a focused button or field keeps
its own Space/Enter, and after a row button acts focus returns to the row, so
`Space` then peeks. An action needing
more than the subject opens its dialog in one click. Acting on a row removes it
only when the action takes it out of the queue's filter (toast:
`presentation.success`, else "<label> done"); a row that still matches stays
and updates in place. Each act settles against the server: a row the server
still returns comes back, a failure rolls back, and **Undo** shows only while
the request is in flight — it un-hides the row and does not reverse the action.
A queue is only as good as its data: model status enums with `valueColors` and
aging timestamps.
