---
title: Chapter 5 — Compose a ViewSpec
description: Use view_specs.declare to shape how the app renders — table, detail, panel, button, composite, metric, file, and board. A ViewSpec shadows the renderer auto-default for an ObjectType and is validated against the live registry at declare time.
---

# Chapter 5 — Compose a ViewSpec (`view_specs.declare`)

The final chapter shapes how your app **renders**. A ViewSpec shadows the
renderer's auto-default for an ObjectType. (ViewSpecs are rendered by the
Integrated-tier web app; the declaration itself is identical on every
surface.) The declaration is validated against the **live registry** at
declare time — unknown ObjectTypes, source mismatches, and bad column refs
are rejected before anything persists.

> `Customer.list` etc. are illustrative; substitute your own ObjectTypes.

## The action

| | |
|---|---|
| **Action** | `view_specs.declare` |
| **Input** | `{ name: string, object_type: string, spec: ViewSpec }` |
| **Envelope** | `{ data: { name, registered: true }, approval_status: 'not_required', audit_event_id }` |
| **Idempotency** | `per-key` keyed on `name` — re-declaring is a deliberate override |

- **`name`** — dot-separated segments, e.g. `Customer.list`,
  `Customer.detail`. Case-sensitive; the resolver keys on the bare string.
- **`object_type`** — the ObjectType this view targets; must be registered.
- **`spec`** — the view body, a discriminated union on `type`.

## The component palette

A closed set of nine variants. Every variant accepts an optional `title`
(section-chrome heading above the block) and an optional `showWhen` predicate
list (render only when all predicates hold).

| `type` | Renders | Key fields |
|---|---|---|
| `table` | row-per-record list (or row-per-group with an aggregate source) | `source`, `columns[]`, `title?` |
| `detail` | one record's property grid | `source`, `properties?`, `title?` |
| `panel` | a form hosting an ActionType (mutations) | `actionForm`, `title?` |
| `button` | a single action trigger | `action`, `presetParams?`, `label?`, `title?` |
| `composite` | nested specs with a `layout` axis | `children[]`, `layout?`, `title?` |
| `metric` | one scalar KPI tile (aggregate without `groupBy`) | `source`, `label`, `format?`, `title?` |
| `file` | an image / attachment surface | `source` (attachments or file-property), `title?` |
| `board` | a kanban board grouping records by an enum property | `source`, `groupBy`, `columns?`, `rowActions?`, `title?` |
| `queue` | a prioritized action worklist (keyboard-first triage) | `title`, `why`, `source`, `order`, `row`, `actions` |

**`composite` layout axis** — `layout` is one of `"stack"` (default, vertical),
`"grid"` (responsive columns; children may be `{ span: 1–4, view: <spec> }`
wrappers), or `"tabs"` (each child's `title` becomes its tab label — tabs
children MUST declare `title`).

**`button.label`** is the button control caption (text on the button itself).
`title` is a separate section heading above the button.

**Every variant is a CLOSED shape** — the schema decodes with
`onExcessProperty: 'error'`, so a field it doesn't declare is REJECTED, not
ignored. Author to the exact keys: sort a table via **`source.sort`** (`"-name"`
= descending), NOT a top-level `sort`/`limit`; a block's heading is **`title`**,
not `label` (`label` is only a metric caption, a button caption, or a column-ref
hint `{ ref, label }`); and a `composite` scopes a child to its parent through
the child's **`source.filter`** (e.g. `{ customer_id: "$context.id" }`), never a
field on the composite itself.

## Example A — a table over `customer`

`spec.source.objectType` must agree with the top-level `object_type`. Every
column ref must resolve to a property or link on that ObjectType.

```json
{
  "name": "Customer.list",
  "object_type": "customer",
  "spec": {
    "type": "table",
    "source": { "objectType": "customer" },
    "columns": ["name", "email", "status", "deals.count"]
  }
}
```

Column refs can traverse links and aggregate them:

- `deals.count` — `count` is the only **property-less** aggregate.
- `deals.sum:amount` — `sum`/`avg`/`min`/`max` take a `:property`
  argument that must exist on the link target and (for `sum`/`avg`) be
  numeric.
- `customer` (a bare link) — the renderer picks a sensible default.

## Example B — a scalar metric tile

A `metric` is the same aggregate shape used **without** a `groupBy` — one
number. Its `source.objectType` may be any ObjectType the org owns (a tile
can summarise something other than the declared `object_type`).

```json
{
  "name": "Customer.active_count",
  "object_type": "customer",
  "spec": {
    "type": "metric",
    "label": "Active customers",
    "source": { "objectType": "customer", "aggregate": { "op": "count" }, "filter": { "status": "active" } }
  }
}
```

## Example C — a panel that hosts an action

A `panel` binds an ActionType (e.g. one you declared in Chapter 4) so the
app can render a mutation form. The bound action must be registered.

```json
{
  "name": "Deal.send_proposal_panel",
  "object_type": "deal",
  "spec": {
    "type": "panel",
    "actionForm": { "action": "deal.send_proposal" }
  }
}
```

## Invoke it — three surfaces

**CLI:**

```sh
canon actions invoke view_specs.declare --input @customer-list.json
```

**REST** (`{ input }` wraps `{ name, object_type, spec }`):

```sh
curl -X POST https://api.canonify.app/v1/actions/view_specs.declare \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "name": "Customer.list",
      "object_type": "customer",
      "spec": {
        "type": "table",
        "source": { "objectType": "customer" },
        "columns": ["name", "email", "status"]
      }
    }
  }'
```

**MCP** — the `actions.invoke` tool:

```json
{
  "name": "actions.invoke",
  "arguments": {
    "name": "view_specs.declare",
    "args": {
      "name": "Customer.list",
      "object_type": "customer",
      "spec": { "type": "table", "source": { "objectType": "customer" }, "columns": ["name", "email", "status"] }
    }
  }
}
```

## Re-declaring is the override path

`view_specs.declare` is `idempotency: 'per-key'` keyed on `name`.
Re-declaring under the same `name` with a different `spec` is the
deliberate override — the stored spec is replaced, re-validated against
today's schema, and any prior "broken" flag is reset.

## Failure modes

| Symptom | `error.details.code` | Recovery |
|---|---|---|
| `object_type` not registered | `unknown_object_type` | declare/refine the ObjectType first (Chapters 1–2) |
| `spec.source.objectType` disagrees with `object_type` | `source_object_type_mismatch` | make them match (except `metric`, which may summarise any ObjectType) |
| A column ref doesn't resolve | `unknown_column_ref` | the property / link / aggregate must exist on the ObjectType |
| `panel`/`button` binds an unknown action | `unknown_action` | declare the ActionType first (Chapter 4) |
| `source.sort` names a property that doesn't exist | `unknown_column_ref` | sort key must be a real property on the ObjectType |
| `groupBy` on a `metric` source | `invalid_metric_group_by` | `metric` is scalar-only — remove `groupBy` |
| `file` block's property doesn't exist or isn't file-typed | `invalid_file_source` | the property must exist and be `file` type on the ObjectType |
| `tabs` composite child missing `title` | `invalid_tabs_child` | every `tabs` layout child must declare a `title` (tab label) |
| `span` on a child outside a `grid` composite | `invalid_composite_span` | `{ span, view }` wrappers are only valid under `layout: "grid"` |
| `format: "currency"` or `"percent"` on a non-numeric column | `invalid_column_format` | currency/percent require a numeric property; date requires a timestamp |
| `table` `rowGroups.by` names a non-existent property | `invalid_row_groups_by` | `by` must be a real property on the ObjectType |
| `rowGroups` declared on an aggregate-backed `table` | `row_groups_on_aggregate` | row grouping sections per-record rows — use the aggregate source's `groupBy` instead |

## Preview before you declare

`view_specs.preview` is a read-only dry-run that resolves a spec against
live data and reports, per block, what the renderer would show. Use it
between drafting and declaring to catch blank columns and missing actions
early.

```sh
canon actions invoke view_specs.preview --input - <<'JSON'
{
  "object_type": "customer",
  "spec": {
    "type": "table",
    "source": { "objectType": "customer" },
    "columns": ["name", "email", "status"]
  }
}
JSON
```

The response reports `source.row_count`, per-column `resolved`/`blanked`
status, and action `exists`/`visible` for panels/buttons. An invalid spec
returns `{ valid: false, errors }` instead of throwing, making it safe to
call before `declare`.

**Loop:** draft → preview → fix blanked columns / missing actions → declare.

## Done

You have built an app end to end — tables, a rich typed model, a
lifecycle, custom actions, and rendered surfaces — entirely through
declarations. Verify the whole session with `canon audit --since 1d` and
list your catalog with `canon actions list --json`.

← Back to the [skill overview](../SKILL.md).
