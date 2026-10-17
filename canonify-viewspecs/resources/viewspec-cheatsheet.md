# ViewSpec cheatsheet — start here

One page: every block, every field, the reference grammar, and the ten errors
you will most likely hit. Every shape is **closed** — an unknown or misplaced
field is refused at declare time. The deep pages are linked at the end.

```json
{
  "name": "Job.list",
  "object_type": "job",
  "title": "Jobs",
  "icon": "list",
  "spec": {
    "type": "table",
    "source": { "objectType": "job", "filter": { "status": { "in": ["booked", "ready"] } }, "sort": "-promised_at" },
    "columns": ["title", "customer.name", { "ref": "quote", "label": "Quote" }],
    "rowActions": ["job.start"]
  }
}
```

The declaration envelope is `name` (dot-separated, e.g. `Job.list`),
`object_type` (must equal `spec.source.objectType` for table, detail and
board), `title?` and `icon?` (the nav entry; a Lucide name), `spec`, and for a
re-declare outside a bundle `expectedHash?` or `force?`.

## The ten blocks

Every block also takes `showWhen?` (UX only, never security) and — except
`queue`, where it is required — `title?`, painted once in the block header.

| `type` | Required | Optional | Shows |
|---|---|---|---|
| `table` | `source`, `columns` | `title`, `rowActions`, `presets`, `page_size`, `rowGroups`, `totals`, `actions`, `compare`, `window`, `showWhen` | One row per record; with an aggregate `source`, one row per group (then no `totals` / `rowGroups`) |
| `detail` | `source` | `recordLayout`, `properties`, `header`, `links`, `title`, `showWhen` | One record; list → record (master-detail) by default |
| `panel` | `actionForm` | `title`, `showWhen` | An action's form |
| `button` | `action` | `label`, `presetParams`, `sources`, `title`, `showWhen` | A one-tap action |
| `composite` | `children` | `layout`, `filters`, `title`, `showWhen` | Children stacked, in a grid, or as tabs |
| `metric` | `source`, `label` | `format`, `compare`, `window`, `title`, `showWhen` | One figure; with a `@bucket` spine also a sparkline and a delta |
| `chart` | `source` (with `groupBy`) | `kind`, `stacked`, `format`, `compare`, `window`, `partialTrailing`, `title`, `showWhen` | One measure across one dimension |
| `file` | `source` | `title`, `showWhen` | A row's attachments, or one `file` property |
| `board` | `source`, `groupBy` | `columns`, `rowActions`, `createAction`, `overdue`, `lanes`, `actions`, `title`, `showWhen` | Cards in one column per enum value |
| `queue` | `title`, `why`, `source`, `order`, `row`, `actions` | `urgency`, `buckets`, `empty`, `page_size`, `showWhen` | A worklist cleared row by row |

## Every nested shape

| Shape | Fields |
|---|---|
| `source` — records (table, detail, board, queue) | `objectType`, `filter?`, `sort?` (a queue sorts by `order`, never `sort`) |
| `source` — aggregate (table, metric, chart) | `objectType`, `aggregate`, `groupBy?`, `seriesBy?` (needs `groupBy`), `expand?` (the one to-many fan-out), `filter?` |
| `source` — record-scoped metric | `link` (one to-many link of the record in scope), `aggregate` |
| `source` — file | `{ objectType }` (attachments) or `{ property }` (a `file` property) |
| `aggregate` | `op`, `property?`, `of?` (`sum` only: 1–3 numeric factors), `label?` (required with `of` on table and chart) |
| `of` factor | a path, or `{ lookup: { objectType, match, property, default } }` (at most one lookup; `match` has 1–3 keys) |
| `compare` | `offset?: { period, periods }`, `rolling?: { buckets }` — at least one |
| `window` | `last` — the number of trailing buckets |
| column (table, board, detail `properties`, header `metrics`) | a ref string, or `{ ref, label?, sortable?, format?, emphasize? }`; `emphasize` is `[{ when, level }]`, first match wins |
| `rowActions[]` (table, board) | an action name, or `{ action, label?, default?, showWhen?, presetParams?, sources? }` |
| `presets[]` | `{ label, filter }` — a quick filter with a live count |
| `rowGroups` | `by`, `groups?` (`[{ value, label }]`), `defaultCollapsed?` |
| `totals` | names of number / money columns the table shows; summed in a footer |
| `actions` (table, board) | `{ "<action>": { label?, emphasis? } }` — `emphasis` is `primary` |
| `actionForm` (panel) | `action`, `submitLabel?`, `presetParams?`, `sources?` |
| `sources` | `{ "<input>": { action, read } }` — seed an input from another action's result |
| `header` (detail) | `title?`, `subtitle?`, `status?`, `metrics?` (columns) |
| `links[]` (detail) | `{ link, as, limit?, sort?, label? }` — `as` is `tab`, `card` or `chip` |
| `lanes` (board) | `{ "<groupBy value>": { sort?, limit?, create? } }` — a column's own order, cap and '+ New' |
| `overdue` (board) | a boolean property; a passed date is painted late exactly where it is true |
| `children[]` (composite) | a ViewSpec, or `{ span, view }` in a `grid` (`span` is `2` or `4`) |
| `filters[]` (composite) | `{ property, kind, required? }` — one reader filter row that scopes every child |
| `order[]` (queue) | `{ field, dir }` — `dir` is `asc`; the first timestamp drives the age badge |
| `urgency`, `buckets` (queue) | `{ from }` — an enum with colours (card rail) / a date (Overdue · Today · This week · Later) |
| `row` (queue) | `title`, `subtitle?`, `facts?` (≤ 3 × `{ value, label?, format? }`), `peek?` (a `detail` view name) |
| `actions` (queue) | `primary`, `secondary?`, `defer?` — each a name or `{ action, label?, key?, showWhen? }` |
| `empty` (queue) | `title?`, `body?` |
| `showWhen[]` | `{ field, op, value? }` — `op` is `eq`; every entry must hold (an action's own precondition is its `presentation.showWhen`) |

## Grammar

| What | Form | Example |
|---|---|---|
| Column ref | a property, or up to 2 to-one link hops then a property | `"email"`, `"customer.tier"` |
| Link aggregate column | `<to-many link>.count`, or `.<op>:<property>` with `sum` / `avg` / `max` | `"jobs.count"`, `"jobs.sum:revenue"` |
| Grouped-table column | the `groupBy` ref, `count`, or `<op>:<property>` | `"bike.kind"`, `"count"`, `"sum:quote"` |
| `groupBy` | a path, optionally `@day` / `@week` / `@month` / `@quarter` / `@year` | `"status"`, `"collected_at@month"` |
| `seriesBy` | a path, never a `@bucket` | `"bike.kind"` |
| Filter key | a path through to-one links only, or `$as_of` (temporal types) | `"customer.tier"` |
| Filter value | a scalar (equality), `{ gt?, gte?, lt?, lte? }`, or `{ in: [...] }` (≤ 1000) | `{ "quote": { "gte": 100000 } }` |
| Filter literal | typed like the property: booleans `true`/`false`, money in integer minor units, enums a declared value | `{ "is_overdue": true }` |
| Relative date | `$now`, `$now±<n><unit>` (units `s m h d w`; `m` = minutes, no months); `:this_week`, `:this_month` for equality | `{ "due_at": { "lt": "$now+7d" } }` |
| Viewer / record tokens | `$principal` (the viewer's id), `$context.<path>` (the record in scope), `$row.<field>` (a row action's own row) | `{ "owner_id": "$principal" }` |
| `sort` | an own property, `-` prefix for descending | `"-promised_at"` |
| `format` | `currency`, `percent`, `date` (queue facts: `number`, `currency`, `percent`) | money needs no format |
| Emphasis `level` | `good`, `warn`, `danger` | |
| Aggregate `op` | `count` (no property), `sum`, `avg`, `max` | `{ "op": "sum", "property": "quote" }` |
| `compare` | `offset.period` `month` or `year`, `periods` 1–5; `rolling.buckets` 2–12 | needs a `@bucket` spine, `count`/`sum` |
| Chart `kind` | `bar` (default for a category) or `line` (default for a time spine); `stacked` needs `seriesBy` and `bar` | |
| Composite `layout` | `stack` (default), `grid`, `tabs` (every child titled) | |
| Composite filter `kind` | `date-range`, `dimension`, `reference`, `date` | |
| `recordLayout` | `master-detail` (default), `flat` | |
| Queue row text | a property name, or a `$field` template with `{{age <field>}}` | `"$owner · {{age opened_at}}"` |

Colours and labels are declared on the ObjectType, not in the view: enum
`valueColors` (the token table is in the `canonify-objects-actions` skill's
[object-property-types.md](../../canonify-objects-actions/resources/object-property-types.md)),
`valueLabels`, property `label`, money `currency`. A derived property takes
`label`, `values`/`valueLabels` and `currency` too, so a view never restates them.

## The ten most common errors

| Error | Cause | Fix |
|---|---|---|
| `at "<path>": is unexpected, expected: …` | A misspelled field, or one on the wrong level. Inside a composite child the list reads `"type" \| "span" \| "view"`: the field belongs inside the child block. | Check the field against the tables above at that level. |
| `unknown_column_ref` | A column, filter key, `sort`, `showWhen` field or `overdue` names no property of the type. | `canon schema describe --object <type>`; reach other types through a link path. |
| `link_path_to_many` | A filter or column path crosses a to-many link. | Use a link aggregate column (`jobs.count`), or an aggregate `source` with `expand`. |
| `invalid_filter_value_type` | A filter literal does not match the property type. | Booleans `true`/`false`; money in integer minor units (`125000` for 1,250.00); enums a declared value. |
| `source_object_type_mismatch` | `object_type` differs from `spec.source.objectType` on a table, detail or board. | Make them equal; a dashboard composite may carry any `object_type`. |
| `unsupported_sortable_column_ref` | `sortable: true` on a dotted path. | Sort by an own property, or denormalize the value onto the type. |
| `invalid_chart_group_by`, `invalid_metric_group_by` | A chart with no `groupBy`, or a metric grouped by a category. | A chart needs `groupBy`; a metric is a scalar or a `@bucket` spine. |
| `invalid_aggregate_compare`, `invalid_aggregate_window` | `compare` / `window` without a `@bucket` spine, over `avg`/`max`, with `seriesBy`, or with a filter on the spine property. | Group by `<timestamp>@month` (or another bucket), use `count` or `sum`, scope time with `window`. |
| `measure_label_required` | `aggregate.of` on a table or chart has no `label`. | Add `aggregate.label`. |
| `unknown_action`, `unresolvable_subject` | A bound action is not declared, or a row / queue / board action has no input that carries the row id. | Declare the action; name its id input `<object_type>_id`, address the row by it in the handler, or declare `subject: { param, object_type }`. |

Also seen: `duplicate_block_heading` (two blocks paint the same heading — give
one a distinct `title`), `invalid_board_lane_create` (a lane's create cannot
land in that column — see the board page), `invalid_queue_row` (a fourth queue
fact), `CONFLICT/HASH_REQUIRED` (re-declaring an existing view outside a bundle
without `expectedHash` or `force`).

## Deep pages

- [viewspec-block-types.md](viewspec-block-types.md) — each block with JSON and what the renderer decides
- [viewspec-binding-and-sources.md](viewspec-binding-and-sources.md) — sources, aggregates, filters, columns, formats, `$context`
- [viewspec-recipes.md](viewspec-recipes.md) — record page, list, dashboard, workflow, forms
- A whole app that applies clean: the `canonify-app-authoring` skill's
  [worked example](../../canonify-app-authoring/resources/worked-example.md)
