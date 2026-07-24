---
title: Starter templates — reach the elevated treatment by default
description: Copy-paste ViewSpec templates that already embody the elevated, well-crafted look — an elegant record detail, a scannable list, a board, and a metric tile — plus a seven-move checklist. Adapt the ObjectType/property names to your app; every field is current vocabulary that validates today.
---

# Starter templates — reach elegance by default

The renderer already does the hard editorial work — currency and dates, FK→title,
status pills, file previews, rollups. What most first-draft views miss is the
**knobs that are already there**: a detail header, `valueColors` on a status,
filter `presets`, a promoted metric, curated columns. This file is a small
library of views that already turn those knobs. **Copy one, rename the
ObjectType and properties to your app, `view_specs.preview` it, then declare.**

Everything here is **current vocabulary** — it validates against today's schema.
Nothing depends on directives that don't exist yet. Where a field is illustrative
(`logo_url`, `renewal_date`, `industry`), swap it for a real property on your
ObjectType; the *shape* is what you're copying.

> These templates lean on two things you declare **on the ObjectType**, not the
> view: `enum` `values` and their `valueColors` (Chapter 2 →
> [objecttype-refinement › *Enum value colors*](objecttype-refinement.md)), and
> the `link`s a record-scoped metric aggregates over. A grey status chip or a
> blank rollup almost always means the ObjectType, not the ViewSpec, needs the
> knob. Fix it at the source.

---

## Reach elegance in 7 moves

A first-draft view is rarely *wrong* — it's flat. These seven moves are the
difference between "renders" and "reads well". Each maps to a finding in the
craft audit ([`viewspec-craft-theming-2026-07.md`](../../../reviews/viewspec-craft-theming-2026-07.md)).
The first three live on the **ObjectType** (declared once, they pay off in every
view); the rest are view-level.

1. **Declare `valueColors` on your status enums.** A bare enum renders as grey
   text; a `valueColors` map (`{ won: cool, lost: hot, … }`) makes it a semantic
   chip *everywhere* — list, detail, board, queue border — for free and
   consistently. (Fixes the "inconsistent status color" gap, C6.)
2. **Add `valueLabels` on those same enums.** A colored chip that still shows the
   raw token (`green`, `proposal_sent`) reads worse than one that shows a human
   label. `valueLabels` (`{ green: Healthy, proposal_sent: "Proposal sent" }`) is
   `valueColors`' sibling — same declaration site, display-only, and it never
   touches filtering (queries still key off the raw value). Declaring colors
   without labels earns the `elegance_missing_value_labels` advisory.
3. **Group and emphasize your actions.** More than a couple of custom actions
   render as a flat button wall. Put related ones under a shared
   `presentation.group` (`"Lifecycle"`, `"Comms"`) so they fold into a menu, and
   mark the one main action `presentation: { emphasis: 'primary' }` so it becomes
   a clear inline CTA. (Fixes the ungrouped-action wall the
   `elegance_ungrouped_action_wall` advisory flags.)
4. **Give every `detail` a `header`.** `{ title, subtitle, status, avatar }`
   promotes the record's primary field to a real heading and lifts the status
   out of the property grid — instead of a flat wall of identical `dt`/`dd`
   rows. (Fixes flat hierarchy, C4.)
5. **Add `presets` to busy lists.** One-tap filter chips (`Open`, `This week`,
   `My deals`) beat making the operator learn the filter bar — the common cuts
   are one tap, declared, not user-remembered.
6. **Promote a metric.** Surface the rollup an operator would otherwise sum by
   hand — a record-scoped `{ link, aggregate }` metric ("Lifetime spend",
   "Orders") right under the header. (Fixes "rollups render nowhere", C1.)
7. **Curate columns.** Pick 4–6 that matter, set `width`/`format`, and
   `emphasizeWhen` the one cell that carries the signal. A 12-column dump reads
   as noise; a curated row reads as a decision. (Fixes flat one-weight lists, C4.)

The templates below are these moves, already made.

---

## Template 1 — an elegant record detail

The elevated-detail pattern: a **header band** (title/subtitle/status/avatar), a
**metric strip** of record-scoped rollups, a **curated property grid** with the
primary field promoted out of it, and **one prioritized linked section** — not
twelve stacked ones. It's a `composite` of three children: `detail` → a grid of
`metric` tiles → a scoped `table`.

```json
{
  "name": "Customer.detail",
  "object_type": "customer",
  "title": "Customer",
  "icon": "building-2",
  "spec": {
    "type": "composite",
    "layout": "stack",
    "children": [
      {
        "type": "detail",
        "source": { "objectType": "customer" },
        "header": {
          "title": "name",
          "subtitle": "industry",
          "status": "status",
          "avatar": "logo_url"
        },
        "properties": [
          { "ref": "owner", "label": "Account owner" },
          "email",
          "phone",
          { "ref": "renewal_date", "label": "Renews", "format": "date" }
        ]
      },
      {
        "type": "composite",
        "layout": "grid",
        "title": "At a glance",
        "children": [
          {
            "type": "metric",
            "source": { "link": "orders", "aggregate": { "op": "sum", "property": "total" } },
            "label": "Lifetime spend",
            "format": "currency"
          },
          {
            "type": "metric",
            "source": { "link": "orders", "aggregate": { "op": "count" } },
            "label": "Orders"
          },
          {
            "type": "metric",
            "source": { "link": "deals", "aggregate": { "op": "sum", "property": "amount" } },
            "label": "Open pipeline",
            "format": "currency"
          }
        ]
      },
      {
        "type": "table",
        "title": "Recent orders",
        "source": {
          "objectType": "order",
          "filter": { "customer_id": "$context.id" },
          "sort": "-created_at"
        },
        "columns": [
          { "ref": "created_at", "label": "Date", "format": "date" },
          "status",
          { "ref": "total", "label": "Total", "format": "currency" }
        ],
        "page_size": 5
      }
    ]
  }
}
```

```
+---------------------------------------------------------------+
|  [logo]  Acme Offices ApS                        ( Active )   |  <- header: title + status chip
|          Facilities · Copenhagen                              |     subtitle
+---------------------------------------------------------------+
|  Account owner  Ada Lovelace     Email  ops@acme.test         |  <- curated grid (primary field
|  Phone  +45 12 34 56 78          Renews  3 Sep 2026           |     lives in the header, not here)
+---------------------------------------------------------------+
|  At a glance                                                  |
|  +----------------+ +-------------+ +---------------------+   |  <- record-scoped metric strip
|  | Lifetime spend | | Orders      | | Open pipeline       |   |     ({ link, aggregate })
|  |  kr 482.000    | |     37      | |   kr 96.000         |   |
|  +----------------+ +-------------+ +---------------------+   |
+---------------------------------------------------------------+
|  Recent orders                                                |  <- ONE prioritized linked section
|  Date         | Status     | Total                            |     (scoped to $context.id)
|  2 Jul 2026   | fulfilled  | kr 7.320                         |
|  18 Jun 2026  | fulfilled  | kr 4.110                         |
+---------------------------------------------------------------+
```

**Why this reads well.** The header gives the record a real identity line — the
name is a heading, not a `dt`/`dd` row, and the status is a chip lifted out of
the grid (its color comes from the `status` enum's `valueColors`, declared on the
ObjectType). The `avatar` slot degrades gracefully: a real `logo_url` renders the
logo, otherwise the ObjectType's type icon (a building for a company) — never a
fake initials square on an organisation (leading-glyph principle). The metric
strip answers "how much does this customer matter?" from **rollups you'd
otherwise sum by hand** — a one-hop `{ link, aggregate }` reads the current
record's related rows, so it needs no filter. And there is exactly **one** linked
section, scoped and sorted newest-first, instead of a wall of "Show 0 records"
tables. Curate to the section that earns the space.

> **Record-scoped metric shape.** `{ "link": "orders", "aggregate": … }` is the
> one-hop form — it aggregates the *current record's* linked rows. `link` names a
> single link (a two-hop `customer.orders` is rejected), and a `sum`/`avg`
> property must exist and be numeric on the link target. See
> [viewspec-analytics › *Record-scoped metric*](../../canonify-viewspecs/resources/viewspec-analytics.md).

---

## Template 2 — a scannable list

A list an operator can *scan*, not just read: curated columns with `width` and
`format`, a value column that `emphasizeWhen`s on the row that matters, a status
column that's a colored chip (from the enum's `valueColors`), and `presets` for
the cuts people actually take.

```json
{
  "name": "Deal.list",
  "object_type": "deal",
  "title": "Deals",
  "icon": "table",
  "spec": {
    "type": "table",
    "source": { "objectType": "deal", "sort": "-amount" },
    "columns": [
      { "ref": "name", "label": "Deal", "sortable": true, "width": "lg" },
      "stage",
      {
        "ref": "amount",
        "label": "Value",
        "format": "currency",
        "sortable": true,
        "width": "sm",
        "emphasizeWhen": [{ "field": "stage", "op": "eq", "value": "won" }]
      },
      { "ref": "close_date", "label": "Close", "format": "date", "width": "sm" },
      "owner"
    ],
    "presets": [
      { "label": "Open", "filter": { "stage": { "in": ["open", "proposal_sent"] } } },
      { "label": "Won this month", "filter": { "stage": "won", "close_date": ":this_month" } },
      { "label": "My deals", "filter": { "owner": "$principal" } }
    ],
    "page_size": 25
  }
}
```

```
+----------------------------------------------------------------------+
|  Deals     [ Open ] [ Won this month ] [ My deals ]      [ filter ▾ ] |  <- one-tap presets
+---------------------+-----------+-----------+-----------+-------------+
|  Deal               | Stage     |   Value   |  Close    | Owner       |
+---------------------+-----------+-----------+-----------+-------------+
|  Northwind renewal  | ( Won )   | kr 96.000 | 2 Jul 26  | Ada         |  <- Value emphasized on won
|  Contoso expansion  | (Proposal)| kr 45.000 | 14 Aug 26 | Ben         |
|  Fabrikam pilot     | ( Open )  | kr 12.000 | 30 Sep 26 | Ada         |
+---------------------+-----------+-----------+-----------+-------------+
```

**Why this reads well.** The name column is wide and sortable — the key column
carries visual weight instead of every column being the same "Rhea" density. The
`Stage` chip is colored because the `stage` enum declares `valueColors` on the
ObjectType — so the same status reads the same everywhere, not a chip in the list
and grey text in the detail. `emphasizeWhen` puts a single, semantic highlight on
the `Value` cell exactly when a deal is `won`, so the eye lands on closed money
without a rainbow of ad-hoc colors. And `presets` turn the three cuts an operator
actually takes ("open", "won this month", "mine") into one-tap chips — `:this_month`
is a curated calendar token and `$principal` auto-scopes to the caller, so no id
is hard-coded. The result is a table you triage by scanning, not by reading every
cell.

> **Column `width` tokens.** `xs | sm | md | lg | xl`, or a CSS length/fraction
> (`120px`, `2fr`, `25%`, `8ch`). `format` is `currency | percent | date` and is
> checked against the property type at declare time.

---

## Template 3 — a board and a metric tile

### 3a — a board with good defaults

A kanban grouped by a status enum, with curated card fields, an inline row
action, and a create control at the board and per-column level.

```json
{
  "name": "Deal.pipeline",
  "object_type": "deal",
  "title": "Pipeline",
  "icon": "kanban",
  "spec": {
    "type": "board",
    "source": { "objectType": "deal", "sort": "-amount" },
    "groupBy": "stage",
    "columns": [
      { "ref": "name", "label": "Deal" },
      { "ref": "amount", "format": "currency" }
    ],
    "rowActions": ["deal.advance"],
    "createAction": "deal.create"
  }
}
```

```
+  open  ------+  + proposal_sent +  +  won  ------+  +  lost  -----+
| + new        |  | + new          |  | + new       |  | + new       |
+--------------+  +----------------+  +-------------+  +-------------+
| Fabrikam     |  | Contoso        |  | Northwind   |  |             |
| kr 12.000    |  | kr 45.000      |  | kr 96.000   |  |             |
|   [Advance]  |  |   [Advance]    |  |             |  |             |
+--------------+  +----------------+  +-------------+  +-------------+
```

**Why this reads well.** The board groups on the same `stage` enum whose
`valueColors` color the chip everywhere else, so the columns read as the pipeline
an operator already has in their head. Cards are curated to two fields — the name
and the money — instead of dumping every property, so a column of cards stays
scannable. `createAction` adds "+ new" at the board *and* per column (the
per-column control presets the new card's `stage` to that column), and
`rowActions` puts the one move that matters — `advance` — inline on the card. The
subject for `deal.advance` resolves from the `deal_id` convention; if your action
names its id param differently, declare `subject: { param, object_type }` on the
ActionType or apply fails with `unresolvable_subject` (better than a dead button).

### 3b — a metric tile with good defaults

A scalar KPI: an aggregate with a `filter`, a clear `label`, and a `format` so the
number renders as money, not a bare comma-grouped integer.

```json
{
  "name": "Deal.won_revenue",
  "object_type": "deal",
  "title": "Revenue (won)",
  "icon": "trending-up",
  "spec": {
    "type": "metric",
    "source": {
      "objectType": "deal",
      "aggregate": { "op": "sum", "property": "amount" },
      "filter": { "stage": "won" }
    },
    "label": "Revenue won",
    "format": "currency"
  }
}
```

```
+----------------------------+
|  Revenue won               |
|                            |
|        kr 1.240.500        |
+----------------------------+
```

**Why this reads well.** A metric is worth declaring when a single number is the
answer — here, closed revenue. The `filter` scopes the aggregate (`stage: won`),
`format: "currency"` gives it a currency glyph and grouping instead of a raw
`1240500`, and the `label` names it plainly. When a number has a natural ceiling —
seats used vs. licensed, quota attainment — reach for the meter variant instead:
add `"display": "gauge"` (or `"bar"`) and a `"target"` (a literal, or a
number-yielding aggregate) and the tile renders as a filled meter. A row of these
over a grouped `table` is the canonical dashboard
([viewspec-analytics › *KPI dashboard composite*](../../canonify-viewspecs/resources/viewspec-analytics.md)).

---

## Before you declare — preview

Every template above is a full `view_specs.declare` payload. Adapt the names, then
run it through the read-only dry-run first so a typo'd column fails *before* it
persists:

```sh
canon actions invoke view_specs.preview --input @customer-detail.json
```

The report is per-block: `columns[].status` is `resolved` or `blanked`,
`source.row_count` shows how many rows would render, and an invalid spec comes
back as `{ valid: false, errors }` instead of throwing. Fix any `blanked` column,
then `view_specs.declare` the same payload. Full loop in
[Chapter 5 — Compose a ViewSpec](viewspec-quick-start.md).

## See also

- [Chapter 2 — Refine the ObjectType](objecttype-refinement.md) — where `enum`
  `values`, `valueColors`, and `valueLabels` (the status colors + labels these
  templates rely on) are declared.
- [Chapter 5 — Compose a ViewSpec](viewspec-quick-start.md) — the full palette,
  failure modes, and the declare/preview loop.
- [`canonify-viewspecs`](../../canonify-viewspecs/SKILL.md) — the block-by-block
  reference (every field, every source shape) behind these templates.
- [`viewspec-craft-theming-2026-07.md`](../../../reviews/viewspec-craft-theming-2026-07.md)
  — the craft audit these moves answer.
