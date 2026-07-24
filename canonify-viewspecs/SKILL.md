---
name: canonify-viewspecs
description: Author Canonify ViewSpecs — the declarative way to build a screen. Compose catalog pieces (ObjectTypes + ActionTypes) into a transport-agnostic view from 10 block types (table, detail, panel, button, composite, metric, chart, file, board, queue), then declare it to an org with view_specs.declare.
version: 1.0.0
---

# Canonify ViewSpecs — building screens without code

A **ViewSpec** is a screen described as data, not code. You don't write
React, HTML, or CSS. You hand the platform a small JSON document that says
*"show this ObjectType as a table with these columns"* or *"put a form for
this Action in a panel"*, and the renderer turns it into a real screen.

Think of it like ordering from a menu instead of cooking. The kitchen (the
renderer) already knows how to make every dish; you just pick the dishes and
say how to plate them. The dishes are the **10 block types** below. The
ingredients are the things already in your catalog — your **ObjectTypes**
(your typed data shapes) and your **ActionTypes** (the things a user can do).

A ViewSpec never names a database table, a SQL query, or a raw API URL. It
names *catalog pieces by their declared names*, and the platform resolves
them at render time. That's what makes a ViewSpec **transport-agnostic**: the
same `{ type: "table", source: { objectType: "customer" }, columns: [...] }`
describes the data whether you read it back as text on the CLI, as JSON over
REST, or rendered as a real table in the Integrated-tier web app. The block
is a *description of intent*; each surface fulfils it its own way.

> **Where ViewSpecs render.** The full visual rendering (the shadcn tables,
> the metric tiles, the action forms) is the **Integrated tier** — the
> `apps/web` app. The JSON spec itself, and the data it binds to, are
> surface-neutral: an agent on the CLI or REST reads the same ObjectType
> rows and the same aggregate figures the web renderer reads. You author one
> spec; the tier decides how rich the pixels get.

If this is your first ViewSpec, start by looking at what you already have to
bind to:

```sh
canon objects list-types          # the ObjectTypes you can point a source at
canon actions list                # the Actions you can wire to panels/buttons
canon schema describe --object customer   # one ObjectType's properties + links
```

Everything below is how to turn those pieces into a screen.

---

## 1. What a ViewSpec is

A ViewSpec is a **discriminated union** — one JSON object whose `type` field
picks which kind of block it is. The renderer (`apps/web`'s
`ViewSpecRenderer`) reads `type` and dispatches to the matching renderer.
Every block also carries an optional `showWhen` for conditional display
(more on that in section 4).

The smallest possible ViewSpec is a single block:

```json
{
  "type": "table",
  "source": { "objectType": "customer" },
  "columns": ["name", "email"]
}
```

That's a complete, valid screen: a table of every `customer`, two columns.
The renderer fills in everything else — column headers from the ObjectType's
labels, sensible formatting per property type, pagination, the filter bar.

A real screen is usually several blocks stacked together. You do that with
the `composite` block, which holds a list of `children` (each child is
itself a full ViewSpec). Composites nest, so you can build a page out of a
metric row on top, a master table in the middle, and a detail panel below —
all in one document.

The shapes are **closed**. Every block type rejects unknown fields at
declare time (`onExcessProperty: 'error'`). If you misspell `colums`, the
platform tells you instead of silently ignoring it.

---

## 2. The 10 block types

There are exactly **ten** block types. Memorise this table; the rest of
the skill is a deep-dive on each. Full field-by-field reference with a JSON
example and an ASCII layout for every one lives in
[resources/viewspec-block-types.md](resources/viewspec-block-types.md).

| `type` | What it shows | Binds to | Key fields |
|---|---|---|---|
| `table` | A list — one row per record, or one row per group | `ObjectTypeSource` *or* `AggregateSource` | `source`, `columns` (≥1), `rowActions?`, `presets?`, `page_size?`, `title?` |
| `detail` | One record's properties as a labelled grid | `ObjectTypeSource` | `source`, `properties?`, `header?`, `title?` |
| `panel` | A form for an Action, generated from its input schema | an `actionForm` binding | `actionForm`, `title?` |
| `button` | A one-tap Action trigger | an Action by name | `action`, `presetParams?`, `label?`, `title?` |
| `composite` | A container with a `layout` axis, optionally with a reader filter bar | nothing (holds `children`) | `children` (≥1), `layout?`, `filters?`, `title?` |
| `metric` | A single scalar KPI tile (e.g. "Revenue: 42k"), a meter, or a view's `"hero"` lead figure — optionally with a delta badge and a `trend` sparkline | `AggregateSource` (no `groupBy`) *or* a record-scoped `{ link, aggregate }` | `source`, `label`, `format?`, `display?`, `target?`, `goodDirection?`, `compare?`, `trend?`, `title?` |
| `chart` | A bar or line chart of one measure across one dimension | `AggregateSource` (**with** `groupBy`) | `source`, `kind?`, `format?`, `title?` |
| `file` | An image or download link | a row's attachments (`{ objectType }`) *or* a `file` property | `source`, `title?` |
| `board` | A kanban board grouping records by an enum property | `ObjectTypeSource` | `source`, `groupBy`, `columns?`, `rowActions?`, `createAction?`, `title?` |
| `queue` | A prioritized action worklist — keyboard-first triage, inline row actions | `ObjectTypeSource` | `title`, `why`, `source`, `order`, `row`, `actions` (`primary` req), `urgency?`, `buckets?`, `empty?`, `page_size?` |

A few things worth internalising:

- **`table` is two blocks in one.** If its `source` has an `aggregate`, it's
  a *grouped summary table* (one row per `groupBy` value). Otherwise it's a
  plain record list. The renderer dispatches on the presence of `aggregate`.
- **`metric` reuses the table's aggregate shape.** A metric is just an
  `AggregateSource` used *without* `groupBy` — one number instead of one row
  per group. That's deliberate: the same aggregate vocabulary backs both, so
  a KPI tile and a grouped table can never disagree on what `sum:amount`
  means.
- **`panel` vs `button`.** A panel renders the whole *form* for an Action
  (every input field). A button is the single-tap version for an Action you
  can fire with no extra input (or with `presetParams` filling everything).
- **`board` for kanban layouts.** Groups an `ObjectTypeSource` into columns
  by a required `groupBy` enum property. Per-card fields use the same
  `ColumnRef` grammar as `table`; omit `columns` to fall back to the
  ObjectType's primary properties. Use `rowActions` (an array) to surface
  per-card action triggers. Set `createAction` (an ActionType, usually
  `<type>.create`) to add '+ New' at the board and per-column level — the
  per-column control presets the new card's group. If a screen needs bespoke UI that no palette
  block covers, that is a Hosted/External-tier concern — the Tier-1 palette
  grows first-class variants on pull.
- **`queue` for action worklists.** The default landing for action-oriented
  apps: a prioritized, keyboard-first triage list where each row carries only
  the decision-context an operator needs and the actions are inline. `title`
  and `why` (the "why now" one-liner) are both required; `order` IS the
  urgency (no separate priority field); `actions.primary` is the action a row
  is cleared with. It is **additive** — it sits alongside `table`/`board`/
  `detail` for browse/report/bulk, not instead of them — and it raises the
  data-model bar (model status enums + aging timestamps for the "why now"
  signal). Full field reference, keyboard map, and a worked bundle in
  [resources/viewspec-block-types.md §9](resources/viewspec-block-types.md).
- **`title` — section chrome on every block.** Every variant accepts an
  optional `title` string rendered as a section heading above the block.
  On `button`, `label` is the button control caption (text on the button
  itself); `title` is the separate section heading. On `metric`, `label` is
  the required tile caption; `title` is the optional heading above it.
- **`composite` layout axis.** `composite` accepts an optional `layout`:
  `"stack"` (default, vertical), `"grid"` (responsive columns, with optional
  `{ span: 1–4, view: <ViewSpec> }` wrappers for width), or `"tabs"` (each
  child's `title` becomes its tab label — tabs children require `title`).
  A `composite` also accepts an optional `filters` axis — a reader-adjustable
  filter row (`{ property, kind: "date-range" | "dimension" }`) that scopes
  *every* child to the same slice so the figures always agree.
  Full examples in
  [resources/viewspec-composition-patterns.md](resources/viewspec-composition-patterns.md).

---

## 3. Binding — pointing a block at the catalog

A block is inert until you bind it to a source. There are two source shapes,
both of which name an ObjectType and resolve against the live registry at
render time. Full reference in
[resources/viewspec-binding-and-sources.md](resources/viewspec-binding-and-sources.md).

**ObjectTypeSource** — "show these records":

```json
{
  "objectType": "customer",
  "filter": { "tier": "gold", "created_at": { "gte": "$now-30d" } },
  "sort": "name"
}
```

Each `filter` entry is an equality scalar (`"gold"`), a range predicate over
`gt` / `gte` / `lt` / `lte`, a set-membership `{ "in": [...] }` (SQL `IN`; an
empty array is a never-match), or a not-equal `{ "neq": ... }` (SQL `<>`). A
value of `"$principal"` (alias `"$actor"`) resolves at render time to the
calling principal's id (auto-scope a "my records" view); an unresolved token is
dropped. Range bounds may be a literal or a **relative-date token** (`$now-30d`).
Supported units are `s`, `m`, `h`, `d`, `w` (`m` = minutes; months are *not*
supported). An **equality** filter may also use a curated calendar token — one
of `:today`, `:yesterday`, `:this_week`, `:this_month`, `:last_7d`, `:last_30d`
(e.g. `{ "created_at": ":this_week" }`), resolved server-side to a date window
and rejected at decode if it isn't in that closed set. Every filter key must be a real property on the ObjectType — a typo
is rejected at declare time with `unknown_column_ref`. `sort` names a property
(`"name"` for ascending; `"-name"` for descending) and is also validated at
declare time.

**AggregateSource** — "reduce these records to a figure":

```json
{
  "objectType": "deal",
  "aggregate": { "op": "sum", "property": "amount" },
  "groupBy": "stage"
}
```

The five operations are `count`, `sum`, `avg`, `min`, `max`. `count` takes
**no** property (it counts rows); the other four **require** one. `sum` and
`avg` additionally require that property to be *numeric*. Add `groupBy` to
get one row per group (a grouped table); leave it off to get one scalar (a
metric).

**Columns** use a compact dot-traversal grammar. A bare property is just its
name (`"email"`). You can reach across a link and aggregate it inline:
`"appointments.count"` (count of linked rows) or `"orders.sum:amount"` (sum
of a numeric property on the linked ObjectType). When you need a label, sort
hint, format, or conditional emphasis, swap the string for an object:
`{ "ref": "amount", "label": "Total", "sortable": true, "format": "currency"
}`. `format` is `"currency"` | `"percent"` | `"date"` (validated at declare
time for property-type compatibility). `emphasizeWhen` highlights cells when
predicates hold — same grammar as `showWhen`. Full column grammar in
[resources/viewspec-binding-and-sources.md](resources/viewspec-binding-and-sources.md).

**The render context (`$context.*`).** A `detail` block needs to know *which*
record to show, and a child table inside a composite needs to scope to its
parent's row. Both read from the render context with the `$context.<path>`
grammar. A detail resolves its record id from `$context.<objecttype>.id`
(falling back to `$context.id`); a scoped filter uses
`{ customer_id: "$context.id" }`. `presetParams` on a button or panel use the
same grammar to pre-fill an Action input from the row in scope.

---

## 4. Recipes — composing beautiful views

Real screens are compositions. Four recipes cover almost everything; each has
its own resource with full JSON and an ASCII layout you can copy.

**Master-detail** — a list that drills into a record. A `composite` holds a
`table` (the master) and a `detail` (driven by `$context`). See
[resources/viewspec-master-detail.md](resources/viewspec-master-detail.md).

**KPI dashboard** — a row of `metric` tiles over a grouped `table`. Mix
`count` and `sum:property` aggregates; group the table by a status property.
See [resources/viewspec-analytics.md](resources/viewspec-analytics.md).

**Forms and actions** — a `panel` for the create/edit form plus `button`
blocks for one-tap transitions, with `presetParams` pulling from
`$context`. See
[resources/viewspec-forms-and-actions.md](resources/viewspec-forms-and-actions.md).

**Nested composites and modals** — 2–3 levels of `composite` nesting to lay
out a full page, plus the `showWhen` pattern for conditionally-revealed
sections (e.g. a refund panel that only appears for paid orders). See
[resources/viewspec-composition-patterns.md](resources/viewspec-composition-patterns.md).

**`showWhen` — conditional display.** Every block accepts an optional
`showWhen`: an array of predicates (the *same* `{ field, op, value? }`
grammar Canonify forms use, with `op` ∈ `eq` / `in` / `neq` / `present`). The
block renders only when **all** predicates hold. One critical caveat, worth
repeating because it bites people: `showWhen` is **UX, not security**. It
hides the block in the UI; it does **not** stop the data being fetched, and
it runs client-side. Never use it to withhold sensitive data — that's the
job of server-side policy + column projection.

**Declaring and applying a view.** A ViewSpec on its own is just a document.
To attach it to an org so the renderer picks it up, wrap it in a
`view_specs.declare` call:

```json
{
  "name": "Customer.list",
  "object_type": "customer",
  "spec": {
    "type": "table",
    "source": { "objectType": "customer" },
    "columns": ["name", "email"]
  }
}
```

`name` is a dot-separated handle (e.g. `Customer.list`, `Deal.pipeline`).
`object_type` must match the spec's `source.objectType` for `table` and
`detail` blocks (a mismatch is rejected with `source_object_type_mismatch`).

Two optional SIBLINGS of `spec` (siblings, not keys inside the spec body — the
body schema rejects excess keys) shape how the View appears in the nav:

- **`title`** — the human label shown instead of the machine `name`
  (e.g. `"Pipeline"`). Absent → the web humanizes `name`.
- **`icon`** — a [Lucide](https://lucide.dev) icon name (e.g. `"kanban"`,
  `"users"`, `"layout-dashboard"`). The navbar renders Views as the product's
  primary menu (icon + label rows, no "Views" header), so this is the menu-item
  icon. Absent → a default icon. A canon build SHOULD declare one on every View
  (the Pax generator fails the build otherwise).

```json
{
  "name": "Deal.pipeline",
  "object_type": "deal",
  "title": "Pipeline",
  "icon": "kanban",
  "spec": {
    "type": "board",
    "source": { "objectType": "deal" },
    "groupBy": "stage"
  }
}
```

`metric` sources may point at any ObjectType the org owns. The
action validates everything against the live registry — unknown ObjectType,
unknown column, unknown bound Action all fail loudly — then UPSERTs into the
org's `_view_specs` keyed by `name`. Re-declaring the same `name` overrides
it. This is **isomorphic**: the exact same call works on all three surfaces.

```sh
# CLI
canon actions invoke view_specs.declare --input @customer-list.json
```

```
# REST: POST /v1/actions/view_specs.declare   body = { name, object_type, title?, icon?, spec }
# MCP : actions.invoke tool  { name: "view_specs.declare", args: { name, object_type, title?, icon?, spec } }
```

---

## 5. Preview before you ship

Before declaring a spec to the org, use **`view_specs.preview`** to see what
the renderer *would* show — without writing anything. It is the
declare–preview–iterate loop that catches blank columns and missing actions
early.

Input is EITHER `{ name }` (preview a declared view by name) OR
`{ object_type, spec }` (inline dry-run, no prior `declare` needed):

```sh
canon actions invoke view_specs.preview --input - <<'JSON'
{
  "object_type": "customer",
  "spec": {
    "type": "table",
    "source": { "objectType": "customer" },
    "columns": ["name", "email", "loyalty_tier"]
  }
}
JSON
```

The report is per-block. For each block you get:

| Field | Meaning |
|---|---|
| `source.row_count` | ObjectTypeSource: one COUNT(*) probe — how many rows would render |
| `source.value` / `source.group_count` | AggregateSource: the scalar or the number of groups |
| `columns[].status` | `resolved` — the column ref resolves; `blanked` — it would render empty |
| `actions[].exists` / `.visible` | panel/button/board rowActions — does the action exist? can the caller see it? |
| `context_refs[]` | `$context.*` refs in `presetParams` that need a record in scope |
| `show_when[].field_exists` | whether the `showWhen` predicate field exists on the ObjectType |

An inline dry-run that fails declare validation returns `{ valid: false, errors }` instead of throwing — the report is the feedback channel.

**The loop:**

1. Draft the spec.
2. `view_specs.preview { object_type, spec }` — check for `blanked` columns,
   missing actions, zero `row_count` where rows are expected.
3. Fix the spec; repeat until all columns are `resolved` and counts look right.
4. `view_specs.declare` — persist the validated spec.

---

## 6. Next steps

You now have the whole model: 10 block types, two source shapes, a small
column + filter grammar, `showWhen` conditioning, one declare action that
applies a finished view to an org over any surface, and `view_specs.preview`
to verify before shipping.

A sane authoring loop:

1. **Discover** what you can bind to: `canon objects list-types`,
   `canon actions list`, `canon schema describe --object <type>`.
2. **Start with one block**, get it declaring cleanly, then wrap it in a
   `composite` and add blocks.
3. **Preview each iteration.** `view_specs.preview { object_type, spec }` tells
   you which columns resolve and how many rows the source would return — before
   you commit anything.
4. **Declare incrementally.** Each `view_specs.declare` is idempotent on
   `name`, so re-running with a tweaked spec just overrides — tight feedback.
5. **Read back over a different surface** to confirm isomorphism: declare via
   CLI, then `GET /v1/objects/...` for the rows the table binds to. Same data
   either way.
6. **Test accessibility.** Because the Integrated tier renders real DOM, run
   the web accessibility checks (the `web-design-guidelines` / accessibility
   review pass) against the rendered screen: keyboard navigation through
   tables and forms, label associations on panel inputs, colour-contrast on
   metric tiles, and a logical heading order across composite sections. A
   ViewSpec that validates against the schema can still ship an inaccessible
   screen — the schema guards shape, not usability.

Deep-dive resources:

- [resources/viewspec-block-types.md](resources/viewspec-block-types.md) — every block, field by field
- [resources/viewspec-binding-and-sources.md](resources/viewspec-binding-and-sources.md) — sources, columns, filters, `$context`
- [resources/viewspec-master-detail.md](resources/viewspec-master-detail.md) — list → record drill-down
- [resources/viewspec-analytics.md](resources/viewspec-analytics.md) — metrics + grouped tables
- [resources/viewspec-forms-and-actions.md](resources/viewspec-forms-and-actions.md) — panels, buttons, `view_specs.declare`
- [resources/viewspec-composition-patterns.md](resources/viewspec-composition-patterns.md) — nesting + the cross-transport apply path
