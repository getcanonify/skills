# Recipes

Copy one, rename the ObjectType and properties to yours, run it through
`view_specs.preview`, then declare. Every recipe is current vocabulary; field
rules are in [viewspec-block-types.md](viewspec-block-types.md). A whole app that
applies clean is the `canonify-app-authoring` skill's
[worked-example.md](../../canonify-app-authoring/resources/worked-example.md).

## Seven moves from "renders" to "reads well"

The first three live on the ObjectType and pay off in every view.

1. **`valueColors` on status enums** — the lifecycle pill and coloured dots
   everywhere (list, detail, board, chart, queue rail) instead of grey text.
2. **`valueLabels` on the same enums** — `proposal_sent` reads "Proposal sent";
   filtering still keys off the raw value.
3. **Group and promote actions** — `presentation.group` folds related actions
   into a menu; `presentation.emphasis: "primary"` makes the main one a clear
   inline button.
4. **A `header` on every `detail`** — the title is a heading and the status a
   pill, not rows in the grid.
5. **`presets` on busy lists** — the cuts people take (`Open`, `This week`,
   `Mine`) are one tap.
6. **Promote a metric** — the rollup an operator would sum by hand, as a
   record-scoped `{ link, aggregate }` tile under the header.
7. **Curate columns** — four to six that matter, `format` where needed, and one
   `emphasize` rule on the cell that carries the signal (it paints that
   property on every surface of the type).

A grey pill or a blank rollup almost always means the ObjectType, not the view,
lacks the declaration.

## Record page — header, metric strip, one linked section

```json
{
  "name": "Customer.detail", "object_type": "customer", "title": "Customer", "icon": "building-2",
  "spec": { "type": "composite", "children": [
    { "type": "detail", "source": { "objectType": "customer" },
      "header": { "title": "name", "subtitle": "industry", "status": "status" },
      "properties": [{ "ref": "owner", "label": "Account owner" }, "email", "phone", { "ref": "renewal_date", "label": "Renews", "format": "date" }] },
    { "type": "composite", "layout": "grid", "title": "At a glance", "children": [
      { "type": "metric", "source": { "link": "orders", "aggregate": { "op": "sum", "property": "total" } }, "label": "Lifetime spend" },
      { "type": "metric", "source": { "link": "orders", "aggregate": { "op": "count" } }, "label": "Orders" } ] },
    { "type": "table", "title": "Recent orders", "page_size": 5,
      "source": { "objectType": "order", "filter": { "customer_id": "$context.id" }, "sort": "-created_at" },
      "columns": [{ "ref": "created_at", "label": "Date", "format": "date" }, "status", "total"] } ] }
}
```

For an ordinary list → record surface, a standalone `detail` with `links` is
enough — the rail is derived from its source.

## Scannable list

```json
{
  "name": "Deal.list", "object_type": "deal", "title": "Deals", "icon": "table",
  "spec": { "type": "table", "page_size": 25, "source": { "objectType": "deal", "sort": "-amount" },
    "columns": [
      { "ref": "name", "label": "Deal", "sortable": true }, "stage",
      { "ref": "amount", "label": "Value", "sortable": true,
        "emphasize": [{ "when": [{ "field": "stage", "op": "eq", "value": "won" }], "level": "good" }] },
      { "ref": "close_date", "label": "Close", "format": "date" }, "owner" ],
    "presets": [
      { "label": "Open", "filter": { "stage": { "in": ["open", "proposal_sent"] } } },
      { "label": "Won this month", "filter": { "stage": "won", "close_date": ":this_month" } },
      { "label": "My deals", "filter": { "owner": "$principal" } } ] }
}
```

## Dashboard — tiles, a reader filter row, charts

KPI tiles go in their own `grid` composite so they stay two-up on a phone; the
outer composite's `filters` scope every child to the same slice, so the tiles
and charts always agree. A `compare` tile (a sparkline and delta, see the
`metric` block) may not put its own `source.filter` on its spine property
(`invalid_aggregate_compare`); time-scope it with `window` instead.

```json
{
  "type": "composite",
  "filters": [{ "property": "created_at", "kind": "date-range" }, { "property": "stage", "kind": "dimension" }],
  "children": [
    { "type": "composite", "layout": "grid", "children": [
      { "type": "metric", "source": { "objectType": "deal", "aggregate": { "op": "count" } }, "label": "Deals" },
      { "type": "metric", "source": { "objectType": "deal", "aggregate": { "op": "sum", "property": "amount" } }, "label": "Revenue" } ] },
    { "type": "composite", "layout": "grid", "children": [
      { "span": 2, "view": { "type": "chart", "source": { "objectType": "deal", "aggregate": { "op": "sum", "property": "amount" }, "groupBy": "created_at@month" }, "title": "Revenue by month" } },
      { "span": 2, "view": { "type": "table", "source": { "objectType": "deal", "aggregate": { "op": "count" }, "groupBy": "stage" }, "columns": ["stage", { "ref": "count", "label": "Deals" }, { "ref": "sum:amount", "label": "Value" }] } } ] }
  ]
}
```

## Workflow — a lifecycle board and a release queue

A board shows where everything is; a queue is where one role clears its work.
Give the type a state machine, a boolean `is_overdue` (derived) and status
`valueColors` first.

```json
{
  "name": "Analysis.outgoing", "object_type": "analysis", "title": "Outgoing", "icon": "inbox",
  "spec": { "type": "queue", "title": "Ready to release", "why": "Finished reports, longest waiting first.",
    "source": { "objectType": "analysis", "filter": { "status": "complete" } },
    "order": [{ "field": "completed_at", "dir": "asc" }],
    "row": { "title": "title", "subtitle": "$client_id · {{age completed_at}}", "facts": [{ "value": "fee", "label": "To invoice" }] },
    "actions": { "primary": { "action": "analysis.release", "label": "Release", "key": "Enter" } },
    "empty": { "title": "Nothing waiting" } }
}
```

`analysis.release` moves the row out of the `complete` filter, so it leaves the queue.
For the board, use the block reference's `board` example (`overdue`, a
`released` lane capped at five, `createAction: "analysis.run_now"`); a
create-then-transition "Run now" lands its card straight in its target column.

## Forms — a record with an edit form and a conditional action

```json
{ "type": "composite", "children": [
  { "type": "detail", "source": { "objectType": "order" }, "properties": ["reference", "total", "status"] },
  { "type": "panel", "actionForm": { "action": "order.update", "submitLabel": "Save" } },
  { "type": "panel", "actionForm": { "action": "order.refund", "submitLabel": "Issue refund" },
    "showWhen": [{ "field": "status", "op": "eq", "value": "paid" }] } ] }
```

Both panels sit under a record, so `order_id` auto-binds — no `presetParams`.
The refund panel appears only on a paid order (a "modal" is just a panel gated
by `showWhen`); the server still governs who may refund. Put field order,
defaults and picker scopes on the action's `presentation`, so every surface
that opens it shows the same form.
