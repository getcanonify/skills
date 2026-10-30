# ViewSpec cheatsheet — start here

One page: every block, every field, the reference grammar, and the ten errors
you will most likely hit. Every shape is **closed** — an unknown or misplaced
field is refused at declare time. The deep pages are linked at the end.

```json
{
  "name": "Analysis.list",
  "object_type": "analysis",
  "title": "Analyses",
  "icon": "list",
  "spec": {
    "type": "table",
    "source": { "objectType": "analysis", "filter": { "status": { "in": ["queued", "complete"] } }, "sort": "-due_at" },
    "columns": ["title", "client.name", { "ref": "fee", "label": "Fee" }],
    "rowActions": ["analysis.start"]
  }
}
```

The declaration envelope is `name` (dot-separated, e.g. `Analysis.list`), `object_type` (must equal
`spec.source.objectType` for table, detail, board and calendar), `title?` and `icon?` (the nav entry; a
Lucide name), `spec`, and for a re-declare outside a bundle `expectedHash?` or `force?`.

## The eleven blocks

Every block also takes `showWhen?` (UX only, never security) and — except
`queue`, where it is required — `title?`, painted once in the block header.

| `type` | Required | Optional | Shows |
|---|---|---|---|
| `table` | `source`, `columns` | `title`, `rowActions`, `presets`, `page_size`, `rowGroups` (+ `groupActions`), `peek` (a detail view's name), `totals`, `actions`, `compare`, `window`, `primary` (`true`: the page's work list, its count leads the page summary; absent, the first list holding rows), `showWhen` | One row per record; with an aggregate `source`, one row per group (then no `totals` / `rowGroups`) |
| `detail` | `source` | `recordLayout`, `properties`, `header`, `totals`, `links`, `files`, `title`, `showWhen` | One record; list → record (master-detail) by default |
| `panel` | `actionForm` | `title`, `showWhen` | An action's form |
| `button` | `action` | `label`, `presetParams`, `sources`, `title`, `showWhen` | A one-tap action |
| `composite` | `children` | `layout`, `filters`, `title`, `showWhen` | Children stacked, in a grid, or as tabs |
| `metric` | `source`, `label` | `format`, `compare`, `better`, `window`, `context`, `title`, `showWhen` | One figure (or `{ difference \| sum: [a, b] }` / `{ ratio: { of, over } }` of two aggregates); with a `@bucket` spine also a sparkline and a delta, else a context line (`context: { text?, metric }` or `{ breakdown: { by, limit? } }`); `compare: "previous_period"` + `better: "up" \| "down"` for the change vs the window before |
| `chart` | `source` (with `groupBy`) | `kind`, `stacked`, `format`, `compare`, `window`, `partialTrailing`, `limit`, `title`, `showWhen` | One measure (or a list of 2–8, one series each) across one dimension; `limit` is a number (one dashed line) or a path (`route.capacity_kg`: each bar its own) |
| `file` | `source` | `title`, `attach`, `showWhen` | A row's attachments (`attach: false` lists them without the Attach picker — UI only, the API still attaches), or one `file` property |
| `board` | `source`, `groupBy` | `columns`, `rowActions`, `createAction`, `overdue`, `lanes`, `subGroupBy`, `mute`, `measure`, `groupActions`, `window` + `moveAction` (a date `groupBy`), `actions`, `primary`, `title`, `showWhen` | Cards in one column per enum value, or per day / week |
| `calendar` | `source`, `start` (a timestamp), `end` or `duration` (minutes), `resource` (the to-one link that picks the column), `hours` (`{ from, to }`, `"HH:MM"`) | `resources` (`{ source }`: which columns, in order), `slot` (15 / 30 / 60), `columns`, `available` (`{ source, resource, weekday, from, to }` — working hours as `time`s), `unavailable[]` (`{ source, resource?, from, to?, label? }` — dates or instants; no `resource` shades every column), `capacity` (`{ by, of, source? }`), `presets` (narrow the events), `createAction`, `moveAction`, `title`, `showWhen` | A day on the org's wall clock, a column per resource (or Week: one resource, a column per working day), `slot`-minute rows, each event at its start for its length in its status colour, the shading, a strip per `capacity.by` row against `of`, the now line, ‹ Today › and a date; a free slot opens `createAction` with the start and resource preset, a drop runs `moveAction` (refused: shown); a phone shows the picked column as an agenda: its events and a Book row per free gap |
| `queue` | `title`, `why`, `source`, `order`, `row`, `actions` | `urgency`, `buckets`, `empty`, `page_size`, `primary`, `showWhen` | A worklist cleared row by row |

## Every nested shape

| Shape | Fields |
|---|---|
| `source` — records (table, detail, board, calendar, queue) | `objectType`, `filter?`, `sort?` (a queue sorts by `order`, never `sort`) |
| `source` — aggregate (table, metric, chart) | `objectType`, `aggregate`, `groupBy?`, `seriesBy?` (needs `groupBy`), `expand?` (the one to-many fan-out), `filter?` |
| `source` — record-scoped metric | `link` (one to-many link of the record in scope), `aggregate` |
| `source` — file | `{ objectType, include? }` (the attachments of the record in scope — that type; `include` adds the files of the records across those links, a to-one parent or to-many children) or `{ property }` (a `file` property) |
| `aggregate` | `op`, `property?`, `of?` (`sum` only: 1–3 numeric factors), `label?` (required with `of` on table and chart); a chart may take a list of 2–8 |
| `of` factor | a path, or `{ lookup: { objectType, match, property, default } }` (at most one lookup; `match` has 1–3 keys) |
| `compare` | `offset?: { period, periods }`, `rolling?: { buckets }` — at least one |
| `window` | `last` (trailing buckets ending now) or `range` (a calendar token; `:next_7d` reaches forward) |
| column (table, board, detail `properties`, header `metrics`) | a ref string, or `{ ref, label?, sortable?, format?, emphasize?, of? }`; `emphasize` is `[{ when, level }]`, first match wins; `of` (table, header `metrics`) names the row's limit for `ref` (a share bar) |
| `rowActions[]` (table, board) | an action name, or `{ action, label?, default?, showWhen?, presetParams?, sources? }` |
| `groupActions[]` (board, a `rowGroups` table) | `{ action, label?, when? }` — on a group's header, its key(s) preset; a row meeting `when` offers it; counted by its `confirm.reads` count |
| `presets[]` | `{ label, filter, default? }` — a quick filter with a live count; the one `default: true` is the view's opening filter |
| `rowGroups` | `by`, `groups?` (`[{ value, label }]`), `defaultCollapsed?`, `measure?` (`{ ref, of? }`), `subGroups?` (one nested level, same keys) — each section counts and pages its own rows |
| `totals` | names of number / money columns the table shows; summed in a footer |
| `actions` (table, board) | `{ "<action>": { label?, emphasis?, hidden? } }` — `emphasis` is `primary`; `hidden: true` withdraws an inferred offer from this view |
| `actionForm` (panel) | `action`, `submitLabel?`, `presetParams?`, `sources?` |
| `sources` | `{ "<input>": { action, read } }` — seed an input from another action's result |
| `header` (detail) | `title?`, `subtitle?`, `status?`, `metrics?` (columns), `stepper?: false` (hides the state-machine stepper) |
| `totals` (detail) | `[ref \| { ref, label?, sign?: -1, always?: true }]` — number field / rollup / one-hop aggregate lines; the last is the total |
| `links[]` (detail) | `{ link \| related: { path }, as, limit?, sort?, label?, primary?: true }` — `as` is `tab`, `card` or `chip`; `path` is at most two to-many links |
| `files` (detail) | `{ include: [<link>, …] }` — the record's files plus those across the links, under its fields and in a queue peek |
| `lanes` (board) | `{ "<groupBy value>": { sort?, limit?, create? } }` — a column's own order (its cards show the sort field), cap and create (titled by the action's label) |
| `overdue` (board) | a boolean property; a passed date is painted late exactly where it is true |
| `window` (board) | `{ range, weekdays? }` — a calendar token the lanes open on (‹ › step it); `weekdays` picks `@day` lanes: `mon`, `tue`, `wed`, `thu`, `fri`, `sat`, `sun` |
| `children[]` (composite) | a ViewSpec, or `{ span, view }` in a `grid` (`span` is `2` or `4`) |
| `filters[]` (composite) | `{ property, kind, required?, showWhen? }` — one reader filter row that scopes every child; a filter whose `showWhen` fails is absent |
| `order[]` (queue) | `{ field, dir }` — `dir` is `asc` \| `desc`, each later key orders the ties; a timestamp/date first key is the row's leading time column (an age is a fact: `{{age <field>}}`) |
| `urgency`, `buckets` (queue) | `{ from }` — an enum with colours (row rail) / a date (Overdue · Today · This week · Later) |
| `row` (queue) | `title`, `subtitle?`, `facts?` (≤ 3 × `{ value, label?, format? }`), `peek?` (the name of a `detail` or record view) |
| `actions` (queue) | `primary`, `secondary?`, `defer?` — each a name or `{ action, label?, key?, showWhen? }` |
| `empty` (queue) | `title?`, `body?` |
| `showWhen[]` | `{ field, op, value? }` entries ANDed, or `any_of` / `all_of` groups — the [one condition grammar](../../canonify-objects-actions/resources/actions-handler-patterns.md#conditions--one-grammar-for-every-surface): `op` is `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `in`, `nin`, `contains`, `present` or `absent`; `field` is a property, or on a block or filter the viewer: `$principal.id`, `.kind`, `.roles`, `.groups`. An action's own precondition is its `presentation.showWhen` |

## Grammar

| What | Form | Example |
|---|---|---|
| Column ref | a property, or up to 2 to-one link hops then a property | `"email"`, `"customer.tier"` |
| Link aggregate column | `<to-many link>.count`, or `.<op>:<property>` with `sum` / `avg` / `max` | `"analyses.count"`, `"analyses.sum:revenue"` |
| Grouped-table column | the `groupBy` ref, `count`, or `<op>:<property>` | `"sample.matrix"`, `"count"`, `"sum:fee"` |
| `groupBy` | a path, optionally `@day` / `@week` / `@month` / `@quarter` / `@year` | `"status"`, `"released_at@month"` |
| `seriesBy` | a path, never a `@bucket` | `"sample.matrix"` |
| Filter key | a path through to-one links only, `$as_of` (temporal types), or `$or` / `$and` — a list of filters, nesting ≤ 3 deep | `"customer.tier"`, `{ "$or": [{ "status": "open" }, { "closed_at": { "gte": "$now-30d" } }] }` |
| Filter value | a literal (equality) or `{ <op>: value }` in the [one op set](../../canonify-objects-actions/resources/actions-handler-patterns.md#conditions--one-grammar-for-every-surface); `in` ≤ 1000 | `{ "status": { "neq": "void" } }`, `{ "fee": { "gte": 100000 } }` |
| Filter literal | typed like the property: booleans `true`/`false`, money in integer minor units, enums a declared value | `{ "is_overdue": true }` |
| Relative date | `$now`, `$now±<n><unit>` (units `s m h d w mo y`; `m` = minutes, `mo` = months); `$today`, `$today±<n>d` (an org calendar day); `:today`, `:yesterday`, `:tomorrow`, `:this_week`, `:next_week`, `:this_month`, `:last_7d`, `:last_30d`, `:next_7d`, `:next_30d` (windows: equality = inside, `lte` = before its end); any other `$`-value but `$context.` / the invoker refuses (`unresolved_condition_operand`) | `{ "due_at": { "lt": "$now+7d" } }`, `{ "due": { "lte": "$today" } }` |
| Viewer / record tokens | `$principal` (the viewer's id), `$context.<path>` (the record in scope), `$row.<field>` (a row action's own row) | `{ "owner_id": "$principal" }` |
| `sort` | an own property, `-` prefix for descending; blanks sort last both ways | `"-due_at"` |
| `format` | `currency`, `percent`, `date` (queue facts: `number`, `currency`, `percent`, and for one date property `time`, `date`, `datetime`, `relative`) | money needs no format; `percent` reads a ratio: `0.055` → `5.5%` |
| Emphasis `level` | `good`, `warn`, `danger` | |
| Aggregate `op` | `count` (no property), `sum`, `avg`, `max` | `{ "op": "sum", "property": "fee" }` |
| `compare` | `offset.period` `month` or `year`, `periods` 1–5; `rolling.buckets` 2–12 | needs a `@bucket` spine, `count`/`sum` |
| Chart `kind` | `bar` (default for a category) or `line` (default for a time spine); `stacked` needs `seriesBy` or a measure list, and `bar` | |
| Composite `layout` | `stack` (default), `grid`, `tabs` (every child titled) | |
| Composite filter `kind` | `date-range`, `dimension`, `reference`, `date` | |
| `recordLayout` | `master-detail` (default), `flat` | |
| Queue row text | a property name, or a template of `$field` / `$link.property` refs and `{{age <field>}}` | `"$sample.matrix · {{age received_at}}"` |

Enum `valueColors` / `valueLabels`, property `label` and money `currency` live on
the ObjectType, never in the view ([object-property-types.md](../../canonify-objects-actions/resources/object-property-types.md)).

## What reads first

| You declare | The screen shows ([the model](viewspec-block-types.md#what-reads-first)) |
|---|---|
| the view `title`, `presets`, `emphasize` | the page heading over a derived summary ("120 analyses · 4 overdue") and the primary create |
| a StateMachine (else `display.statusProperty`) | that enum as the row's one pill; every other enum a coloured dot + label |
| `emphasize` on one column | that property in status ink + glyph wherever the type shows it |
| `display.title: "{code} · {title}"` | the code as a muted eyebrow on cards and rows |
| nothing | one value shape per column, chrome that scales, phone rows in fixed slots, record lists stacked, 15rem board lanes, an aligned queue |

## The ten most common errors

| Error | Cause | Fix |
|---|---|---|
| `at "<path>": is unexpected, expected: …` | A misspelled field, or one on the wrong level. Inside a composite child the list reads `"type" \| "span" \| "view"`: the field belongs inside the child block. | Check the field against the tables above at that level. |
| `unknown_column_ref` | A column, filter key, `sort`, `showWhen` field or `overdue` names no property of the type. | `canon schema describe --object <type>`; reach other types through a link path. |
| `link_path_to_many` | A filter or column path (or an action's `inputFilters` key) crosses a to-many link. | Use a link aggregate column (`analyses.count`), or an aggregate `source` with `expand`. |
| `invalid_filter_value_type` | A filter literal does not match the property type. | Booleans `true`/`false`; money in integer minor units (`125000` for 1,250.00); enums a declared value. |
| `source_object_type_mismatch` | `object_type` differs from `spec.source.objectType` on a table, detail, board or calendar. | Make them equal; a dashboard composite may carry any `object_type`. |
| `unsupported_sortable_column_ref` | `sortable: true` on a dotted path (`client.name`, `analyses.sum:fee`). | Declare it on the type and sort by that: a lookup for a linked value, a rollup for a sum over children. |
| `invalid_chart_group_by`, `invalid_metric_group_by` | A chart with no `groupBy`, or a metric grouped by a category. | A chart needs `groupBy`; a metric is a scalar or a `@bucket` spine. |
| `invalid_aggregate_compare`, `invalid_aggregate_window` | `compare` / `window` without a `@bucket` spine, over `avg`/`max`, with `seriesBy`, or with a filter on the spine property; a metric's `"previous_period"` whose reads share no calendar token, or `better` with no `compare`. | Group by `<timestamp>@month` (or another bucket), use `count` or `sum`, scope time with `window`; for `previous_period` give every read the same `:this_week`-style filter. |
| `measure_label_required` | `aggregate.of` on a table or chart has no `label`. | Add `aggregate.label`. |
| `unknown_action`, `unresolvable_subject` | A bound action is not declared, or a row / queue / board action has no input that carries the row id. | Declare the action; declare `"subject": "<input>"` with that input typed `{ "type": "ref <object_type>" }` (or name it `<object_type>_id`). |

Also seen: `duplicate_block_heading` (two blocks paint the same heading — give
one a distinct `title`), `invalid_predicate_operand` (a `value` that does not
fit its `op`), `unknown_column_ref` on a `$principal.*` field in a row
condition (`emphasize`, `rowActions`, queue `actions` — they read the row), `invalid_board_lane_create` (a lane's create cannot
land in that column — see the board page), `invalid_progress_of` / `invalid_board_measure` /
`invalid_chart_limit` / `invalid_metric_terms` / `invalid_metric_ratio` / `invalid_metric_context` (a limit or term in another unit, or a shape no surface draws), `invalid_detail_totals` (a totals line with no figure, or text), `invalid_queue_row` (a fourth queue
fact, or a `row.peek` of another type or block; a table `peek`: `invalid_table_peek`), `detail_source_not_record_in_scope` / `file_source_not_record_in_scope` (a composite's `detail` or attachments block of another type than the record shown — use a child table),
`invalid_calendar_property` / `invalid_calendar_resource` / `invalid_calendar_action` (a calendar role of the wrong time type, a resource that is no to-one link to the columns, a `createAction` without inputs named exactly as the block's `start` and `resource` keys, a `moveAction` without `start`), `CONFLICT/HASH_REQUIRED` (re-declaring an existing view outside a bundle without `expectedHash` or `force`).

## Deep pages

- [viewspec-block-types.md](viewspec-block-types.md) — each block with JSON and what the renderer decides
- [viewspec-binding-and-sources.md](viewspec-binding-and-sources.md) — sources, aggregates, filters, columns, formats, `$context`
- [viewspec-recipes.md](viewspec-recipes.md) — record page, list, dashboard, workflow, forms
- A whole app that applies clean: the `canonify-app-authoring` skill's
  [worked example](../../canonify-app-authoring/resources/worked-example.md)
