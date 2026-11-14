# The 11 block types

One reference per block: its fields, one example, and what the renderer
decides for you (none of those are knobs). Every block also takes `showWhen`
(the [condition grammar](../../canonify-objects-actions/resources/actions-handler-patterns.md#conditions--one-grammar-for-every-surface): entries AND, an `any_of`
group ORs — UX only) and, except `queue`
where it is required, `title`, painted once in the block header. Sources,
filters, columns and the aggregate envelope are in
[viewspec-binding-and-sources.md](viewspec-binding-and-sources.md).

## What reads first

One emphasis per row and per screen, each read from a declaration made once.

- **A page header.** The view's `title` heads the screen over a summary
  derived from counts the page already reads — the total of its work list (the
  one table, queue or board with `primary: true`, else the largest of the view's
  own type holding rows), up to two `presets` whose filter is a danger/warn
  `emphasize` rule's `eq` conditions or one `hot`/`warm` enum value, Overdue
  rows / `overdue` cards, or a dashboard's date and first count metric — with
  that block's create on its right. An empty list beside one with rows folds
  to one line, after the work on a phone. A record route is headed by its record.
- **One pill per row.** A row, card or record shows one pill: the type's
  lifecycle enum (the property a StateMachine governs, else
  `display.statusProperty`, else its first coloured enum). Every other enum is
  a dot and its label in the value's colour, plain text when the value
  declares no colour, muted text for the enum's `neutral` value beside
  coloured ones. Colour is never inferred: declare `valueColors` and
  `valueLabels` on the ObjectType ([the tokens and fallbacks](../../canonify-objects-actions/resources/object-property-types.md#valuecolors--the-token-table)).
- **Emphasis is ink + glyph, declared once.** An `emphasize` rule
  ([grammar](viewspec-binding-and-sources.md#emphasize--semantic-emphasis))
  paints its value in the level's ink behind an alert or clock glyph (`good`
  is ink only), never a tinted box; a date under `danger` or `warn` adds its
  distance from now ("20h late", "4h left") on a muted second line. A holding
  `danger` rule also marks the row with a leading rule where it distinguishes
  that row (not where it holds for every row shown). The rule one column gives
  a property paints that property wherever the type shows it — other lists,
  the record, the board card, the queue fact — unless that surface declares
  its own.
- **One value shape per column.** A table column, queue fact, board card
  field or record metric strip takes one shape from all its values: one date
  in another year puts the year on every date (`27 Apr 2027`, `12 Nov 2026`),
  and whole amounts drop `.00` on every row. A row of tiles abbreviates all
  its figures or none (§6).
- **A code is an eyebrow.** A `display.title` that leads with a plain code
  (`{code} · {title}`) splits on a card and a table row: the code is a muted
  line above the title.

A link ref (`sample.matrix`) or a lookup reads like the target's own property
on every surface, header metrics included: an enum keeps its colours, money
its currency, an FK or `principal` value shows its title ([the reference rule](../../canonify-objects-actions/resources/object-property-types.md#property-kinds)).

## 1. `table` — a list

Required `source`, `columns` (≥ 1). Optional `rowActions`, `presets`,
`page_size`, `rowGroups`, `peek`, `totals`, `actions`, `compare`, `window`. With an
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
  `source.filter` per key (a key both name is replaced) and ANDs across keys
  (a `$or` / `$and` group never replaces one: both must hold).
  Each chip shows the live count its tap would show. `"default": true` (one
  per list, else `duplicate_default_preset`) opens the list on that chip
  unless `?preset=<label>` names another (`?preset=` = none). With no chip
  active the list is the unfiltered `source`; "All" is legal as `{ "label":
  "All", "filter": {} }` — it counts the whole source, never shows active.
- **`rowActions`** — replaces the inferred row actions with a curated list;
  the header keeps the type's create, and an action curated onto the rows is
  not repeated in the header. Each entry is an action name or `{ action,
  label?, default?, showWhen?, presetParams?, sources? }` (`board` takes the
  same grammar). `$row.<field>` and `showWhen` fields are checked against the
  source type; an unknown action is `unknown_action`. A transition of the
  row's type shows only on rows in one of its `from` states, with no
  `showWhen`. This `showWhen` is the view's own condition on the row; the
  action's `presentation.showWhen`
  ([actions-input-governance.md](../../canonify-objects-actions/resources/actions-input-governance.md#presentation))
  is its precondition on every surface — both must
  hold. An entry's caption is its `label`, else the view's `actions`, else
  the owning type's `display.actions`, else `presentation.label`. Board
  cards and queue rows apply the same gates and caption order (a queue has no
  view step). Each row (table, board card, queue) has one inline verb: the `default: true` one, else `emphasis: "primary"`, else
  the first live; of the rest a lone one is inline, two or more go behind `⋯` (a queue's stay inline) — per row after `showWhen` and permissions. Without `rowActions` a generated create / update / delete is never the inline verb: it sits behind `⋯`, and a lone generated update / create gets no `⋯` at all — the row opens its record, where it lives (declare it in `rowActions`, or give it `emphasis: "primary"`, to bring it out). On a row or board card
  a destructive verb — declared `destructive`, or the generated delete — is
  never inline: it sits behind `⋯`, destructive-styled, even alone, unless
  the author gives it `emphasis: "primary"` (`actions` here, or the owner's
  `display.actions`); a record toolbar guards only the generated delete. Without `rowActions`,
  inferred actions drop any action its **owning** ObjectType hides in
  `display.actions`.
- **One tap or a dialog.** An action whose every input is preset by the row
  (subject included) fires on one tap unless it declares
  `presentation.confirm`, is destructive or needs approval; otherwise it opens a
  dialog naming the record (`display.title` · `display.subtitle`) under the
  action (from a row, card, toolbar, queue row or peek, or a record-bound
  `button`), focus on its first (after a failed submit, first invalid) field.
- **`totals`** — names shown `number`/`money` columns of the source type to sum
  in a "Total" footer, formatted like the cells; it sums the whole filtered
  list (a server `sum` while pages remain). Anything else — a rollup too
  ([rollup or derived?](../../canonify-objects-actions/resources/object-links-and-cardinality.md#rollups))
  — is `invalid_table_totals`; on an aggregate table `totals_on_aggregate`.
- **On a record page** a record-scoped table renders like the same table at
  top level: its `columns`, `totals`, sort, `presets`, `rowGroups` and
  `rowActions` are kept, in one card under the section heading. Its header
  offers ONE way to add the child, the most specific: an action about the
  parent that inserts it (`analysis.add_charge` → `charge_line`, discovery
  `creates`; the record bound), else a declared create of the child taking the
  parent FK, else the generated create — the others are not offered unless the
  view's `actions` names them. An action offered there is not repeated in the
  record's toolbar. One that also moves the parent's own state machine
  (`ticket.assign` opening its first task) is a lifecycle verb: it stays on the
  record's toolbar, never in the child header, and still displaces the child's
  creates. In a record composite it is that link's tab (or section), headed by its `title` — so a
  link's columns are picked by declaring this table, never on `links[]`. The
  lists keep child order, and a block declared between the `detail` and the
  first such table (a `file` block) renders between the record body and the
  lists; blocks after them render below.
- **`rowGroups`** — `{ by, groups?, defaultCollapsed?, measure?, subGroups? }`
  sections the rows under collapsible label + count headers, every row kept;
  past one page each section is read on its own (its count reaches its own
  Load more). `groups` labels and orders known values (shown even when empty;
  others follow the enum's order, then the sort); `measure` (`{ ref, of? }`, as
  on a board) sums each section against its limit; `subGroups` nest one level (a
  day, then a route; a third is `row_groups_too_deep`). Errors: `invalid_row_groups_by`
  / `_measure`, `row_groups_on_aggregate`. **`peek`** unfolds a row in place
  into that view of its record, as a queue `row.peek` (`invalid_table_peek`). `groupActions` (§9) head each
  section, taking `by` (a nested one also `subGroups.by`); without `rowGroups`, `invalid_group_action`.
- **`actions`** (table, board) — `{ "<action>": { label?, emphasis?, hidden? } }`
  relabels, promotes (`emphasis: "primary"`) or withdraws an action (`hidden:
  true`: off this view's headers and inferred row menu; a curated `rowActions`
  entry stays) in this view only. Naming an action offers it in a top-level
  table's header (one about the row type stays on the rows), and a report's (a
  table of another type on a view with a metric or chart offers no generated
  create/update/delete unless named); a `primary` one gets its
  own header control, painted filled, instead of sharing `⋯`. Only this
  declared emphasis — the view's `actions` or the type's `display.actions` —
  paints the control filled; an action's own `presentation.emphasis` makes it
  the header's primary offer but leaves it outlined. On a narrow screen a
  header's or record toolbar's primary offer never folds into `⋯` (the other
  controls fold first); it goes icon-only only if it has an `icon`.

**The chrome scales with the content.** A list whose unfiltered total is
≤ 10 rows drops search and `+ Filter` (declared presets stay); a record's
child table does so up to 25 rows. More than two `presets` put search and
presets in a toolbar row (presets one scrolling row, under the search once
they do not fit beside it), and the count reads "Showing N of M" only while
N < M. A table in a `grid` cell beside other blocks is a summary widget: no search, `+ Filter`,
inferred create or inferred row menu (declared `presets`, `rowActions` and
`actions` entries stay). A table whose `source.filter` pins a column to the
viewer (`$principal` / `$actor`: an inbox, "my tasks") offers no inferred
header actions; name one in `actions` to offer it. The title cell leads with a
glyph only when it carries information — the row's own image (an `https` URL in `avatar_url`, `photo_url`, `image_url`,
`logo_url`, `avatar`, `photo`, `image` or `logo`) or initials for a person-like
type. A table fits its card: long text shortens (full value on hover), the
lowest-value columns fold into the title cell (plain text first, primary money
last, never a title field), on a phone into fixed slots (code + pill + `⋯`, title, ≤ 2 meta values,
dot-enums + the money or an emphasized date + the inline row action, its own
line when those fill it); only a lone too-wide title scrolls (`↔`). Derived,
per field for the whole list: an empty value shows `–` in its place.
A rollup (`analyses.sum:revenue`, or one the type declares) is the row's own
figure: it folds like a property, may be the primary money, in its own currency.

**Grouped summary table.** With an aggregate `source`, each column is a
dimension (the `groupBy` path without its `@bucket`, or the `seriesBy` path), a
measure (`count`, `sum:<p>`, `avg:<p>`, `max:<p>`), or the source measure's
bare property; anything else is `invalid_aggregate_table_column`. Measure
columns take `label`/`format` (default "Count", "Sum of Amount") and render in
the order listed; with `seriesBy` or `compare` the table carries only the
source measure. Enum dimensions follow the declared value order and colours.

```json
{
  "type": "table",
  "title": "Work by matrix",
  "source": { "objectType": "analysis", "aggregate": { "op": "count" }, "groupBy": "sample.matrix" },
  "columns": ["sample.matrix", { "ref": "count", "label": "Analyses" }, { "ref": "sum:fee", "label": "Fees" }]
}
```

## 2. `detail` — the record surface

Required `source`; optional `recordLayout`, `properties`, `header`, `totals`,
`links`, `files`.
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
  field prints no row. Fields read label-left; on a wide record the dates and
  money leave the grid for a side card beside it.
- **`header`** — `title`, `subtitle`, `status` (an enum), `metrics`, `stepper`;
  each slot optional, each a ref or link traversal (`organization.cvr`)
  resolved against the linked record; an FK ref (`customer_id`) shows the
  linked record's title. A `subtitle` may be a template of `{ref}` placeholders:
  an enum placeholder shows its value label, an empty one drops with its
  separator (the `display.title` template rule), an unknown ref is
  `unknown_column_ref`. `title` and `status` take one ref, never a template
  (`"{code} {title}"` is `unknown_column_ref`); omit `title` and the heading is
  the record's `display.title`, which may be a `{ template }`
  ([object-property-types.md](../../canonify-objects-actions/resources/object-property-types.md)).
  `metrics` takes column refs (an enum renders as its pill) and IS the record's one strip;
  omitted, one is derived — rollups, then money fields, then link counts — deduped, at most four, never a tile without a value; a list footer omits what the strip shows.
  A status with a state machine draws a stepper of the path the record surely walked (done / current / next): its main path — from the initial state to the state meaning `terminal-good` (a `terminal-bad` or `sink` state is never the goal, and a sink never a step; no terminal-good state, no stepper — declare `meaning` on the machine's states), taking each state's first-declared exit that leads there; colours never change the path, so declare the usual next step first — or — when another route reaches the current state — only the states every route to it passed (a claim paid after a partial decision never shows Approved done), a branch that leads on drawn as a step on its way (Partial → Paid, a hold → back); a branch that leads nowhere (a cancel) is a badge, not a step; a phone reads "Step 3 of 5 · Packed"; `stepper: false` hides it.
- **`totals`** — lines `ref` or `{ ref, label?, sign?: -1, always?: true }`, a card under the key facts, read with the record: each a number field, rollup or one-hop aggregate (`credit_notes.sum:amount`); `sign: -1` subtracts (`−€12.40`), a zero line hides unless `always`, the last is the total (set apart, always shown). Text or no figure is `invalid_detail_totals`.
- **`recordLayout`** — omitted or `"master-detail"` is the record surface.
  `recordLayout: "flat"` is a spatial opt-out only: a flat detail does not opt out of the links contract.
  `links[].as` still presents related records; only the rail, the lifted
  panel and the mobile shell go.
- **`links`** — `{ link, as, limit?, sort?, label?, primary? }` over the type's `cardinality: "many"` links, or `{ related: { path: ["orders", "credit_notes"] }, as, … }`: the rows two to-many links away in one read, each hop under the reader's own read (one it cannot read hides the list; `unknown_detail_link`, `detail_link_not_many`, over two `invalid_related_path`).
  `as` is `tab` (a list the reader works in),
  `card` (a glanceable preview, `limit` rows, default 3) or `chip` (a count in
  the header); `sort` orders the linked rows, `label` renames (columns come
  from a record-scoped table, above). These shape presentation only, never what is readable. There is no implicit `Overview`
  tab: the header, grid, cards and chips stay on screen and full lists hang
  below in declared order, stacked as sections (two or more: the primary — the first, or `primary: true` — shows its page, each other five rows, then "Show more"); only three or more holding
  over 30 rows in all take tabs, landing on the first non-empty one (tabs
  always: a `tabs` composite). Do not add an `Overview` entry.
- **`files`** — `{ include: [<link>, …] }` lists the record's files under its
  fields (and in a queue peek), plus those across each link — a to-one parent
  (its sample's photos) or to-many children — by the
  [`file` block's](#8-file--image-or-download) rules.
- **Declared wins, empty wins.** A declared link with rows keeps its declared
  `as`; a link whose live count is 0 renders as one dimmed `0` chip whatever
  you declared, except a tab in a tab strip, which keeps its place at `0`.
  Once any link is declared — or the view lists the record's children (a record-scoped table, board, queue or calendar,
  at any depth) — every undeclared link is a chip. Otherwise the fallback ranks links by live count: the two highest non-empty
  become full lists (rows newest first by their date), the rest chips — so the layout shifts as data grows: declare `links`.
- **Mobile.** From 900px wide the rail sits beside the panel. Narrower, it is a two-state drill-down:
  the list state (a no-id entry lands here) and the record state (the record URL `/app/<slug>/view/<View>/<id>`; `?id=<id>` redirects there); **Back to list** keeps it.

**Where a row click lands.** A row, announced `Open <title>` (its
`display.title`), opens the type's record view: a view whose
`object_type` is that type and that renders one record of it — a `detail` of
the type, or a `composite` whose first `detail` is of the type. Every view
you declare outranks the page the platform generates for a compound root
(`<type>.list`, flagged `generated: true` in `canon schema describe`), so you
never rename a view to win. Among your declared candidates one no queue only
peeks wins, then a view in the app's nav, then the ascending view name. With none, the generated page; with neither, the auto
record page. With an app active this applies only to types the app's nav
surfaces (a routing rule, not a data boundary). On that route a composite is
one record page: one action toolbar (non-owner details hide theirs), child
tables auto-scoped to the record, header refs traversing links. On a record
page the breadcrumb and a phone's top bar name the record (its
`display.title`), not the view.

**An explicit composite** of a `table` and a `detail` is still accepted for
bespoke rail columns or curated `rowActions` — not to show a code: every rail
row is headed by the record's `display.title` (`{code} · {title}`), announced
`Select <title>`, and offers no inferred actions (the record's toolbar does).
Its other columns fold around the title (the pill, then self-naming values on
a second line, figures at the end); a fact that fits no row's line leaves
every row, lowest value first.

## 3. `panel` — an action form

`actionForm: { action, submitLabel?, presetParams?, sources? }`. The form is
built from the action's input schema and its `presentation` (field order
`formFields`, opening values `inputDefaults`, picker scopes `inputFilters`;
the last two may read the record the form opens from as `$subject.<column>` —
see the `canonify-objects-actions` skill's
[actions-input-governance.md](../../canonify-objects-actions/resources/actions-input-governance.md)),
so every surface that opens the action shows the same form. `title` and
`submitLabel` default to `presentation.label`; without one, no heading and the
localized Submit. A preset input is submitted but not shown — to open the form
on an editable value use `inputDefaults`. A date-time input edits date and time
on the org's clock in a text field in the reader's date format (typing, paste)
with a calendar picker; it submits the UTC instant. `money` shows its currency
and takes major units. A non-nullable enum of 2–4 values, or whole number
`min`–`max` spanning ≤ 10, is a segmented control (wider: a slider); two short
inputs (date, number, money, enum, in `formFields` order) share a wide row. A
failed submit lists the invalid fields up top, each error naming its field.
Submission is the governed invoke.

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
  grid of only single-span `metric`s stays two-up on a phone. A row's last
  child widens into room the next cannot use (a span-2 then a span-4 reads
  4 + 4); declared order is kept. In `tabs` each
  child's `title` is its tab label and is required (`invalid_tabs_child`); a
  child hidden by `showWhen` loses its tab too.
- **Child types.** A child `table`, `metric` or `chart` may bind another
  existing ObjectType than the view's `object_type` (an `invoice_line` table
  under an `invoice` record); a type that does not exist is
  `source_object_type_mismatch`. A child `detail` shows the record in scope
  (the view's `object_type`, or in a stack the first `detail`'s): another type
  is `detail_source_not_record_in_scope` — show those rows as a child table or
  a link, or give the type its own record view.
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
  `date` filters take a typed day (with a calendar button) for a
  date/timestamp path. Paths use the
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

`source` (an aggregate; record-scoped `{ link, aggregate }`, the record in scope's rows over ONE link, never a dotted
path; or two terms), `label` (required), optional `format`, `compare`, `better`, `window`, `context`. Without `groupBy`
it shows one figure. With a `@bucket` spine it reads one bucketed aggregate: the figure is the newest bucket (the one
containing now when the measure zero-fills), `window` draws a sparkline of the newest N, and `compare` a delta: `offset`
the same bucket a period ago (wins), `rolling` the trailing mean ("vs Sep", "vs Sep 1 – 16" for a running bucket; an
em-dash for a null or zero reference; "so far"), `"previous_period"` the bucket before — or, with no spine, the previous
equal window of the one calendar token every read's filter names ("▲ 2.1 pts vs last week": a share moves by points, a
count by its count, else by %). `better: "up" | "down"` inks the change good or bad. Tiles in a row abbreviate all
figures or none (`DKK 23K`). A categorical `groupBy` or `seriesBy` is `invalid_metric_group_by`. Two terms:
`{ difference | sum: [a, b] }` (sales net of credit notes) or `{ ratio: { of, over } }`, the share of two totals as a
percent, "—" over 0 (never a mean of row ratios); each read under the viewer's grants, a
`count`/`sum` in one unit and bucket (`invalid_metric_terms` / `invalid_metric_ratio`, also a ratio of a ratio).
`previous_period` with no shared window (or a `$now` / absolute date bound), or `better` alone: `invalid_aggregate_compare`.

A tile with no spine has one context line after its change (clamped, full text on hover; no value hides it): by
default from the same read (a count's two largest non-`neutral` lifecycle values, a sum's row count). `context` says
why instead: `{ text?, metric }` is a second figure in the source grammar, read under the viewer's grants and written
in its unit ("€2 050 waiting for a decision"); `{ breakdown: { by, limit? } }` splits the tile's own count/sum read by
an enum or text field (`limit` 1–3, default 2: "21 expired · 13 short-dated"). `invalid_metric_context`: a spine tile,
`limit` over 3, a breakdown of terms or an `avg`/`max`, or `by` a number/date.

```json
{ "type": "metric", "label": "Revenue", "window": { "last": 12 }, "compare": { "offset": { "period": "year", "periods": 1 } },
  "source": { "objectType": "order", "aggregate": { "op": "sum", "property": "total" }, "groupBy": "created_at@month" } }
```

```json
{ "type": "metric", "label": "Utilisation this week", "compare": "previous_period", "better": "up", "source": { "ratio": {
  "of": { "objectType": "appointment", "aggregate": { "op": "sum", "property": "minutes" },
          "filter": { "starts_at": ":this_week", "status": { "in": ["booked", "checked_in", "attended", "no_show"] } } },
  "over": { "objectType": "calendar_day", "aggregate": { "op": "sum", "property": "working_minutes" }, "filter": { "date": ":this_week" } } } } }
```

## 7. `chart` — bar or line

`source` with `groupBy` (else `invalid_chart_group_by` — use a metric); optional `kind`, `stacked`, `format`,
`compare`, `window`, `partialTrailing`, `limit`. One measure, split by `seriesBy` — or `aggregate` as a list of 2–8 measures,
one series each off one grouped read (legend: each `label`, else the property's label). A list with `seriesBy`, or
with measures of different units or currencies, is `invalid_chart_measures`; with `compare`,
`invalid_aggregate_compare`.

```json
{
  "type": "chart",
  "source": { "objectType": "analysis", "aggregate": { "op": "count" }, "groupBy": "due_at@month", "seriesBy": "sample.matrix" },
  "title": "Analyses by month, by matrix"
}
```

```json
{ "type": "chart", "kind": "bar", "stacked": true, "title": "Sales by month",
  "source": { "objectType": "order", "groupBy": "placed_at@month", "aggregate": [
    { "op": "sum", "property": "goods_total", "label": "Goods" },
    { "op": "sum", "property": "shipping_total", "label": "Shipping" } ] } }
```

- **`kind`** — inferred: a categorical `groupBy` draws `bar`, a bucketed one
  `line`; set it only to override. An unknown kind fails at decode.
- **`stacked: true`** — segments that compose a total; needs `seriesBy` or a measure list, `bar`
  (set `kind: "bar"` on a time spine) and `count`/`sum`
  (`invalid_chart_stacked`). Without it series are grouped side by side.
- **`format`** — the value axis and tooltip; `currency` takes its code from the
  property, and a property with no currency renders a bare number. A money
  axis names its currency once, on its top tick (`DKK 80K`).
- **`partialTrailing: false`** — opts out of marking the bucket containing now
  (hatched bar, or a dashed final line segment, plus a caption). On a
  categorical spine it is `invalid_chart_partial_trailing`. The read's
  `partial: true` is unaffected; a standalone grouped table never marks it.
- **`compare`** — the prior layer draws dashed in the base hue, the rolling
  mean thin and muted, and the legend turns on.
- **`limit`** — a number draws a dashed line; a path (`account.route.capacity_kg`) is each bar's own limit (its `max`
  per bar, one more read): a tick on the bar, and over it a danger outline with "⚠ 554 / 500". A path limit is refused
  on `stacked`, `line`, a measure list or another unit — kg against g or a plain number (`invalid_chart_limit`).

What the renderer decides: enum dimensions keep their declared order and wear their `valueColors` (series wear them
only when every series gets a distinct one, else `--chart-1..8`); other categories sort by value; time is
chronological. Beyond 12 categories (8 series) the tail folds into **Other** (an `avg` tail is dropped, and the
caption says so). Series take a fixed hue order by identity, never magnitude; a legend names every series in text. A
table view is one click away, with the same split, compare columns and window; a capped read says so in the caption.

## 8. `file` — image or download

`source` is `{ property }` (the one file in a `file` property, read-only) or
`{ objectType }` — every file attached to the record in scope, newest first,
with an **Attach file** button that uploads to it. That `objectType` must be
the record in scope (as for a child `detail`, above): another type is
`file_source_not_record_in_scope` — show its rows as a child table. An unknown
type or a property that is missing or not a `file` is `invalid_file_source`.
`include: [<link>, …]` beside `objectType` also lists the files of the
records across those links — to-many children (a project shows its expenses'
receipts, attached or held in a `file` property) or a to-one parent — each
through that record's own read gates: the first 50 rows per link, saying so
when there are more; a name that is not a link of the record is
`invalid_file_source`. Images show as thumbnails that open full size, others
as download tiles, each named by its filename, grouped under the property or
`attach` slot holding them; one the reader cannot load shows "File unavailable". The picker takes only what
the type's `attachments` rules accept and names them ("Images or PDF, up to
10 MB").
`attach: false` lists the files with no Attach picker (on a `{ property }`
block it is `file_attach_on_property`). It is presentation only: the attach
route still takes a file from anyone the record's `update` admits — limit who
with a policy, and what with the type's `attachments` rules. Uploading and the rules:
[files](../../canonify-isomorphism/resources/runtime-contract.md).

```json
{ "type": "file", "source": { "property": "logo" } }
```

```json
{ "type": "file", "source": { "objectType": "project", "include": ["expenses"] } }
```

## 9. `board` — a kanban

Required `source` (records) and `groupBy` (an enum, or a date `@day` / `@week`);
optional `columns` (card fields; default the primary properties), `rowActions`
(the table grammar), `createAction`, `overdue`, `lanes`, `actions`,
`subGroupBy`, `mute`, `measure`, `groupActions`; on a date `window` (required) and `moveAction`.

```json
{
  "type": "board", "source": { "objectType": "analysis", "sort": "due_at" },
  "groupBy": "status", "columns": ["client_id", "sample_id", "fee"],
  "overdue": "is_overdue", "createAction": "analysis.run_now",
  "lanes": { "released": { "sort": "-released_at", "limit": 5 } }
}
```

- **Columns the data can fill.** An equality or `in` filter on the `groupBy`
  enum limits the columns to the values it admits. Columns share the width
  down to 15rem (16rem under a 48rem board); past six or while they overflow, a
  lane with nothing to work (empty with no create, or a `terminal` state) folds
  to a rail that names, counts and opens it first, then the fewest-card lanes.
- **A time grid.** `"groupBy": "delivery_date@day"` (or `@week`; a date or timestamp) with
  `"window": { "range": ":this_week", "weekdays": ["mon", …] }` (any calendar token) is a lane per day of that
  org-calendar window, in order, one read; only leading empty past days fold; a left-out weekday's cards are counted in the
  header ("+2 on other days"). ‹ Today › move it (kept in the URL, per board); a card dropped on another day runs `moveAction`
  (it takes the date; a time keeps its wall clock). The window owns the date, not `source.filter` (`invalid_board_window`).
- **`subGroupBy`** heads a group per value in each lane (count, totals); **`mute`** (the `showWhen` grammar) fades the
  cards it picks, out of totals. Either, or `measure`, over a column the viewer may not read or sees masked is dropped.
- **`groupActions: [{ action, label?, when? }]`** — an action over one group's rows ("Dispatch route-day"). One taking
  the `subGroupBy` key heads each sub-group, else one taking the lane's (an enum value, a `@day` date) heads the lane; the
  keys are preset, an unmuted row meeting `when` offers it, its handler reads the rows in one `read_many` (≤ 500) under the
  caller's grant, and its confirm counts them only by its `confirm.reads` count. A `@week` or timestamp lane has no one key:
  `invalid_group_action`, as is an action on another type or not taking the key.
- **`measure: { ref, of }`** — each sub-group (else lane) sums `ref` over its unmuted cards against `of`: a number, or
  a path read as its `max` per group in one aggregate per window, both in the units `ref` reads in. "554.3 / 500 kg"
  (the property's `unit`) over a load bar, warn from 80 %, danger over; another unit is `invalid_board_measure`.
- **`lanes`** — `{ "<value>": { sort?, limit?, create? } }`; a lane with `sort`
  or `limit` is read on its own (count badge = the column's total, "Showing N
  of M", Load more). A lane with its own `sort` shows that property on its
  cards (Released, sorted `-released_at`, says when each report went out); a
  lane that is a `terminal` state of the machine on `groupBy` drops the
  deadlines its cards can no longer miss. A key that is not a declared value
  is `invalid_board_lane`.
- **A create only where it lands.** `createAction` adds a board-level create (in the page header when the board is
  the view's root); a column gets its own '+' from `lanes.<value>.create`, else `createAction`, only where that create
  puts the card, labelled by the action (view `actions`, else `display.actions`, else `presentation.label`). Under a
  StateMachine on `groupBy` a plain create lands in its `initial` column, a create-then-transition in its marker's `to`
  (one create per governed column: [patterns.md](../../canonify-app-authoring/resources/patterns.md)); an ungoverned
  enum or a day in any value its input admits (preset). A lane `create` that cannot land is `invalid_board_lane_create`.
- **One labelled move per card**, by the row-verb rule (§1; a transition is live in its `from` states). Solid only
  when declared `emphasis: "primary"`, else outlined.
- **`overdue`** — a boolean property (often a derived `is_overdue`; `invalid_board_overdue` otherwise): a passed date
  paints late exactly where it is true. Without it a passed date paints late on a card whose state still has a move.
- **Card hierarchy** — the heading is the record's `display.title` (a
  leading code is its eyebrow; a column the title already says — its
  property, `via` or a template token — is not repeated; only a type with no
  human title leads with its first column), then one muted `·` line of values,
  a footer of dates, the pill, dot-enums and true flags; an FK column reads as
  the linked record's title text, so name the FK (`sample_id`), not a link
  path. A true flag paints its label (`Repeats`); `false` flags and a `neutral` enum value among coloured ones paint
  nothing — on calendar events, queue rows and folded list facts too; an empty value shows `–` in its place. A lane header sums the card's first money column (not
  on a lane holding fewer cards than its total, e.g. under `limit`).

## 10. `calendar` — a day by resource, or a week of one

Required `source`, `start` (a timestamp), `end` (a timestamp) or `duration` (minutes), `resource` (the to-one FK whose
targets are the columns), `hours` (`{ from, to }`, `"HH:MM"`); optional `resources` (`{ source }`: which columns, in
order), `slot` (15, 30 or 60; default 30), `columns`, `available`, `unavailable`, `capacity` ([their shapes](viewspec-cheatsheet.md#the-eleven-blocks)),
`presets` (a table's: they narrow the events), `createAction`, `moveAction`. Day: a column per resource; Week: a column
per day someone works for the picked one (‹ Today › and a date step them). Events sit at their start for their length in
their status colour (legend; a coloured enum in `columns` overrides), shaded only where declared; a phone shows the
picked column as an agenda (a week day by day): its events and a Book row per free gap (not where every `capacity` strip is full).

```json
{ "type": "calendar", "source": { "objectType": "appointment" }, "presets": [{ "label": "Active", "default": true,
  "filter": { "status": { "neq": "cancelled" } } }, { "label": "Show cancelled", "filter": {} }],
  "start": "starts_at", "end": "ends_at", "resource": "practitioner_id", "hours": { "from": "07:00", "to": "19:00" }, "slot": 15,
  "resources": { "source": { "objectType": "practitioner", "sort": "name" } }, "columns": ["patient_id", "room_id", "status"],
  "available": { "source": { "objectType": "working_hours" }, "resource": "practitioner_id", "weekday": "weekday", "from": "from_time", "to": "to_time" },
  "unavailable": [{ "source": { "objectType": "calendar_day", "filter": { "closed": true } }, "from": "date" }],
  "capacity": { "by": "room_id", "of": "capacity", "source": { "objectType": "room", "sort": "name" } },
  "createAction": "appointment.book", "moveAction": "appointment.move" }
```

A free slot opens `createAction` with the start and column preset; a drop runs `moveAction` with the new start (and the
column if it takes it, else a drop on another column is refused); a handler refusal shows on the block. Their inputs are
named as the block's `start` and `resource` keys (a move needs only `start`), else `invalid_calendar_action`.

## 11. `queue` — a worklist cleared row by row

Required `title`, `why` (the one-line "why now"), `source` (records, no
`sort`), `order`, `row`, `actions`; optional `urgency`, `buckets`, `empty`,
`page_size`, `primary`. The landing for action-oriented work, beside tables and boards.

- **`order`** `[{ field, dir }]` is the urgency; a timestamp or date first key
  leads every row with a time column (`08:30` over `Today`; with `buckets` on
  that field an Overdue row adds `20h late`). An age is a fact (`row.facts`).
- **`urgency: { from }`** — an enum whose `valueColors` paint a rail down the
  row's leading edge (`neutral` paints none); its value is the first fact column.
  **`buckets: { from }`** — a date field grouping rows under Overdue / Today / This week / Later (fixed labels;
  Later takes a missing date too). Past one page each bucket is read on its
  own, so its header counts its rows plus its own Load more.
- **`row`** — `title`, `subtitle?`, `facts?` (≤ 3 `{ value, label?, format? }`,
  each one column aligned across the rows, its `label` the column header; a
  fourth is `invalid_queue_row`), `peek?` (a `detail` of the queue's type, or
  the record view itself — a composite of detail / table / file blocks; unknown
  is `unresolvable_ref`, another type, block or detail type `invalid_queue_row`). A fact's
  `format` is `number` · `currency` · `percent`, or for one date/timestamp
  property named alone (`"$starts_at"`; else `invalid_queue_row`) `time`
  (`09:00`), `date` (`Tue 20 Oct`), `datetime` (`Tue 20 Oct, 09:00`) or
  `relative` (`in 2 days`), on the org's clock. The open peek shows each detail's
  `properties` (no second header, metrics or linked lists), a composite's child
  tables, its files (`files.include` too) and its OTHER actions — never the
  row's own, nor a generated create/update/delete (the record page's). A text is a
  name the type declares — a property, a rollup or a one-hop link ref
  (`client.name`) — which renders empty when the read withholds it (a title
  blanks, a fact shows `–`; the name never prints); any other word is literal.
  Or a template of `$field` and one-hop `$link.property` refs
  (`"$sample.matrix · $received_at"`; an unknown field, link or linked
  property is `unresolvable_ref` at declare) with `{{age <field>}}` — the time since, in
  the reader's units (`now`, `1 min`, `3 days`); a second age:
  `{ "value": "{{age received_at}}", "label": "Waiting" }`. An enum field
  reads its value label, timestamps
  humanize, an empty field drops with its label and one `·` (the
  [template rule](../../canonify-objects-actions/resources/object-property-types.md#display)).
  An FK (`$customer_id`, an `assignee_id`
  fact) renders as the linked record's title. A money fact renders in its
  currency; an enum fact follows the one-pill rule, and a `neutral` value
  among coloured ones leaves its cell empty (the board card's rule).
- **`actions`** — `primary`, `secondary?`, `defer?`, each a name or `{ action,
  label?, key?, showWhen? }`. Per row, an action survives only if its
  `showWhen` holds and, for a transition, the row is in a `from` state; the
  row's primary is the first survivor of `[primary, ...secondary]` and answers
  to the primary's `key`; failing both, a surviving `defer` stands in. Only
  the declared `primary` paints solid; a row led by another verb shows it
  outlined. One queue can hold rows in several states. Each action needs an input
  carrying the row id (`unresolvable_subject`).

```json
{
  "type": "queue",
  "title": "Bench",
  "why": "Every open analysis, soonest due first.",
  "source": { "objectType": "analysis", "filter": { "status": { "in": ["queued", "running", "complete"] } } },
  "order": [{ "field": "due_at", "dir": "asc" }],
  "buckets": { "from": "due_at" },
  "row": { "title": "title", "subtitle": "$client_id · {{age created_at}}", "facts": [{ "value": "fee", "label": "Fee" }] },
  "actions": {
    "primary": { "action": "analysis.start", "label": "Start", "key": "Enter" },
    "secondary": [
      { "action": "analysis.complete", "label": "Mark complete" },
      { "action": "analysis.call_client", "label": "Call", "showWhen": [{ "field": "priority", "op": "eq", "value": "express" }] }
    ]
  },
  "empty": { "title": "Bench clear", "body": "All caught up." }
}
```

How it behaves: every row shows all its buttons; keys `j`/`k` move, `Enter`
primary, `1`–`9` the Nth secondary, `e` defer, `Space` peek, `/` filter, `?`
help, `F` a one-at-a-time focus mode ending in inbox-zero. An action needing more than the subject opens
its dialog in one click. Acting on a row removes it only when the action takes
it out of the queue's filter (toast: `presentation.success`, else "<label>
done"); a row that still matches updates in place. Each act settles against
the server: a row it still returns comes back, a failure rolls back, and
**Undo** shows only while the request is in flight — it un-hides the row and
does not reverse the action.
