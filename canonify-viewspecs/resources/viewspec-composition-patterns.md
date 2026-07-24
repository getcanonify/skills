# Composition and apply

How to nest blocks into a full page, the common composite patterns, and how
to declare + apply the finished ViewSpec to an org identically over REST,
CLI, and MCP.

---

## Nesting with `composite`

`composite` holds a non-empty `children` array; each child is a full ViewSpec,
so composites nest to any depth. The renderer walks children top-to-bottom,
applying each child's `showWhen` as it goes.

A 2-level page: a metric row (itself a composite) on top of a master-detail
(another composite).

```json
{
  "type": "composite",
  "children": [
    {
      "type": "composite",
      "children": [
        {
          "type": "metric",
          "source": { "objectType": "customer", "aggregate": { "op": "count" } },
          "label": "Customers"
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
        }
      ]
    },
    {
      "type": "composite",
      "children": [
        {
          "type": "table",
          "source": { "objectType": "customer", "sort": "name" },
          "columns": ["name", "email", "tier"]
        },
        {
          "type": "detail",
          "source": { "objectType": "customer" },
          "properties": ["name", "email", "tier", "phone"]
        }
      ]
    }
  ]
}
```

```
+----------------------------------------------------------+
| +----------------+  +-----------------------+            |  <- metric composite
| | Customers  128 |  | Revenue (won) € 1.2M  |            |
| +----------------+  +-----------------------+            |
+----------------------------------------------------------+
| +------------------------------------------------------+ |  <- master-detail
| |  Name      | Email          | Tier                   | |     composite
| | ▶ Acme     | ops@acme.test  | Gold                   | |
| +------------------------------------------------------+ |
| +------------------------------------------------------+ |
| |  Customer:  Acme   ops@acme.test   Gold   +45 ...    | |
| +------------------------------------------------------+ |
+----------------------------------------------------------+
```

---

## Arranging children: `layout` (stack | grid | tabs)

`composite` takes an optional `layout` (absent → `"stack"`, the vertical flow
above). The enum is closed — `"stack"`, `"grid"`, `"tabs"` — with no CSS or
breakpoint knobs.

### Dashboard grid

`layout: "grid"` lays children into a responsive grid. A child may be wrapped
`{ "span": 1-4, "view": <ViewSpec> }` to claim that many columns (a wide table
next to two narrow metrics). A plain child takes one column. `span` is only
valid under `"grid"`.

```json
{
  "type": "composite",
  "layout": "grid",
  "children": [
    {
      "type": "metric",
      "source": { "objectType": "customer", "aggregate": { "op": "count" } },
      "label": "Customers"
    },
    {
      "type": "metric",
      "source": { "objectType": "deal", "aggregate": { "op": "sum", "property": "amount" }, "filter": { "stage": "won" } },
      "label": "Revenue (won)",
      "format": "currency"
    },
    {
      "span": 4,
      "view": {
        "type": "table",
        "source": { "objectType": "customer", "sort": "name" },
        "columns": ["name", "email", "tier"]
      }
    }
  ]
}
```

### Tabs

`layout: "tabs"` renders one tab per child; **each child's `title` is its tab
label**, so every tabbed child must declare a `title`. A child hidden by
`showWhen` renders neither its tab nor its panel.

```json
{
  "type": "composite",
  "layout": "tabs",
  "children": [
    {
      "type": "detail",
      "source": { "objectType": "customer" },
      "properties": ["name", "email", "tier"],
      "title": "Overview"
    },
    {
      "type": "table",
      "source": { "objectType": "appointment", "filter": { "customer_id": "$context.id" } },
      "columns": ["scheduled_at", "status"],
      "title": "Appointments"
    }
  ]
}
```

---

## Reader-adjustable filter bar: `composite.filters`

A `composite` accepts an optional `filters` axis: **one filter row above the
children that scopes everything below it**. The author declares *which* of the
children's source properties a reader may adjust; the reader picks a value; that
slice is applied to **every** child — chart, metric tile, table alike — so the
figures on the page always agree with each other.

Each entry is `{ "property": <string>, "kind": "date-range" | "dimension" }`:

- **`kind: "date-range"`** renders a date-window picker over a `timestamp`
  property (the reader narrows the whole dashboard to a time slice).
- **`kind: "dimension"`** renders a value picker over an `enum` property (the
  reader narrows to one category).

```json
{
  "type": "composite",
  "layout": "grid",
  "filters": [
    { "property": "created_at", "kind": "date-range" },
    { "property": "stage", "kind": "dimension" }
  ],
  "children": [
    {
      "type": "metric",
      "source": { "objectType": "deal", "aggregate": { "op": "sum", "property": "amount" } },
      "label": "Revenue",
      "format": "currency"
    },
    {
      "type": "chart",
      "source": {
        "objectType": "deal",
        "aggregate": { "op": "sum", "property": "amount" },
        "groupBy": "created_at@month"
      },
      "kind": "line",
      "title": "Revenue by month"
    },
    {
      "type": "chart",
      "source": {
        "objectType": "deal",
        "aggregate": { "op": "count" },
        "groupBy": "stage"
      },
      "kind": "bar",
      "title": "Deals by stage"
    }
  ]
}
```

```
+------------------------------------------------------------+
|  Date: [ Last 30 days v ]   Stage: [ All v ]               |  <- the filter row
+------------------------------------------------------------+
|  +------------------+  +---------------------------------+ |
|  | Revenue  € 336k  |  |  Revenue by month  (line)       | |  every child
|  +------------------+  |  ___/\__/\_                      | |  reads the SAME
|  +---------------------+---------------------------------+ |  slice, so the
|  |  Deals by stage (bar)   █ █ ▆ ▃                       | |  numbers agree
|  +-------------------------------------------------------+ |
+------------------------------------------------------------+
```

Pick `Last 7 days` and *all three* children re-query that window — the metric,
the monthly line, and the stage bars — so a reader can never see a total that
disagrees with the chart beside it. Choose a `stage` and the same happens for
that dimension. The reader's choice **overlays** (replaces) the named property
on each child that carries it and leaves the child's other filters untouched;
children that don't have the property are unaffected.

**Declare rules** (validated at `view_specs.declare` against the live catalog):

- Every declared `property` must be honorable by **at least one direct child
  source** — the composite's own `object_type` or a child's
  `source.objectType`. A property no child reads is rejected.
- `date-range` requires the property to be a **`timestamp`** on some child;
  `dimension` requires it to be an **`enum`**. A wrong-typed property is
  rejected.
- Either failure is `invalid_composite_filter` (naming the view and the
  offending property).

**Windows the reader can pick — and the current limit.** The date-range picker
offers **All time / Today / Last 7 days / Last 30 days**. Those are the windows
Canonify's relative-date grammar resolves end-to-end, so the figures stay in
agreement across every child. **90-day and custom absolute ranges are not yet
supported** — the relative-date token set stops at `Last 30 days`, and a custom
range would be dropped before it reaches the aggregate, silently disagreeing
with the charts. They are tracked as a follow-up; author date-range filters
knowing the reader picks from the four windows above.

---

## Modal-dialog pattern (conditional panel)

A "modal" is a `panel` (or a composite) gated by `showWhen`. When the gating
field matches, the section reveals; otherwise it never renders.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "detail",
      "source": { "objectType": "order" },
      "properties": ["reference", "total", "status"]
    },
    {
      "type": "panel",
      "actionForm": { "action": "order.refund", "submitLabel": "Issue refund" },
      "showWhen": [{ "field": "status", "op": "eq", "value": "paid" }]
    }
  ]
}
```

```
status = paid    ->   [ Order detail ]
                      [ Refund panel  ]   <- revealed

status = draft   ->   [ Order detail ]
                      (refund panel hidden)
```

---

## Embedded-table pattern (scoped child)

A detail with a related list scoped to the record via `$context.id`. The
child table only shows rows for the row in scope.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "detail",
      "source": { "objectType": "customer" },
      "properties": ["name", "email"]
    },
    {
      "type": "table",
      "source": {
        "objectType": "appointment",
        "filter": { "customer_id": "$context.id" }
      },
      "columns": ["scheduled_at", "status"]
    }
  ]
}
```

---

## Declaring and applying — one call, three surfaces

A finished ViewSpec is applied to an org with the **`view_specs.declare`**
ActionType. Its input is `{ name, object_type, spec }`. The action validates
against the live registry, then UPSERTs into the org's `_view_specs` keyed by
`name`, so re-declaring the same name overrides it.

The payload (`apply.json`):

```json
{
  "name": "Customer.dashboard",
  "object_type": "customer",
  "spec": {
    "type": "composite",
    "children": [
      {
        "type": "metric",
        "source": { "objectType": "customer", "aggregate": { "op": "count" } },
        "label": "Customers"
      },
      {
        "type": "table",
        "source": { "objectType": "customer", "sort": "name" },
        "columns": ["name", "email", "tier"]
      }
    ]
  }
}
```

The **same** declaration is invoked identically on every surface — that's the
isomorphism contract; CI fails if the envelopes diverge:

```sh
# CLI — per-action sugar or the explicit form
canon actions invoke view_specs.declare --input @apply.json
```

```
# REST — the body IS the action input (each ActionType is POST /v1/actions/<name>)
POST /v1/actions/view_specs.declare
Authorization: Bearer <api_key>
Content-Type: application/json

{ "name": "Customer.dashboard", "object_type": "customer", "spec": { ... } }
```

```
# MCP — the actions.invoke tool takes { name, args }
tool: actions.invoke
args: { "name": "view_specs.declare", "args": { "name": "Customer.dashboard", "object_type": "customer", "spec": { ... } } }
```

All three return the same envelope: `{ data: { name, registered: true }, ... }`.

---

## Reading it back (verify isomorphism)

After declaring, confirm the data the view binds to reads back the same over
any surface — declare via CLI, then read the bound rows:

```sh
canon objects list customer --where '{}'   # the rows the table renders
canon actions list                          # confirms view_specs.declare ran
```

```
# REST equivalent
GET /v1/objects/customer
```

Same rows, same shape, whichever surface — the ViewSpec is transport-agnostic
because it only ever names catalog pieces, never a query or a URL.
