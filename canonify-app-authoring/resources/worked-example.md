---
title: Worked example — a complete, small app bundle you can apply
description: A whole Canonify app as one on-disk bundle — three tables, refined ObjectTypes, a state machine, five custom actions, seven views, an app, and a scenario that proves the business loop. Read it, run it with `canon catalog test`, copy it.
---

# Worked example — the Lab app

One complete app as one bundle in `worked-example/`. A testing lab: clients
send in samples, and an analysis of a sample moves through a lifecycle until its
report is released and invoiced. Copy the shape, rename the nouns.

After `canon skills install canonify-app-authoring` it is at
`.agents/skills/canonify-app-authoring/resources/worked-example/`. One file
without installing:
`canon skill --name canonify-app-authoring resource worked-example/canon/actions/analysis.release.json`.

```
worked-example/
  canon/                          # catalog.export layout (no manifest.json, no policies/ — both optional)
    schema/                       # client, sample, analysis
    object-types/                 # enums, links, money, derived properties, display
    state-machines/Analysis.status.json
    actions/                      # analysis.request, .start, .complete, .release, .cancel
    views/                        # dashboard, bench + outgoing queues, board, list, record, client directory
    apps/lab.json                 # nav order, default view, a count badge
  scenarios/lab-loop/session.yaml
```

## Run it

```sh
cd .agents/skills/canonify-app-authoring/resources/worked-example
canon catalog validate canon --strict # 0 errors · 0 warnings
canon catalog test canon scenarios    # no org, nothing durable
```

Expect `"passed": true` and `applied` counts of 3 schema, 3 object types, 5
actions, 1 state machine, 7 views, 1 app. `client.samples` has no `part_of` on
purpose: an analysis references its sample (`analysis.sample`), so a sample is a
first-class record with its own analyses and page, not a part of its client.

To try it in a real org, apply to a **fresh or sandbox org**
(`canon catalog apply canon --force`): apply prunes every managed artifact the
bundle does not mention. To add it to an existing app, copy its files into your
own exported bundle ([catalog-bundle.md](catalog-bundle.md)).

## What each part shows

**Tables.** Money is `INTEGER` minor units (`fee`, `revenue` in øre);
timestamps are ISO `TEXT`; state columns are `TEXT NOT NULL DEFAULT
'<initial>'`, narrowed to enums by the refinement; `revenue` is nullable until
release.

**ObjectTypes.**
- Belongs-to links over every FK (`analysis.sample`, `analysis.client`,
  `sample.client`, `fk_on: source`); parents declare the `many` side + `on_delete`.
- Enums carry `valueLabels` and `valueColors` (`complete` is `good`,
  `cancelled` `hot`).
- Derived properties read like columns: `fee_incl_vat` (derived `money` with
  a currency and a `label`) and `is_overdue` (derived `boolean`, used by the
  board, a dashboard tile and filters).
- `display.title` (`{reference} {site}` for a sample) makes every FK render as a
  title, not an id — table and board cells, record headers, queue rows.
- `analysis` sets `defaults: { create: false, delete: false }`: created only by
  `analysis.request`, cancelled but never deleted; history arrives by import.

**State machine.** `queued → running → complete → released`, plus `cancelled`
from any open state. A state left and re-entered (running ⇄ on hold) is a
[two-way pair](../../canonify-objects-actions/resources/state-machines.md);
an analysis that starts past `queued` (run on arrival) is a
[create-then-transition](patterns.md).

**Actions.**
- `analysis.request` — a create with a rule: `inputFilters` scopes `sample_id`
  to `$input.client_id` (`formFields` client first, `inputDefaults` priority
  `standard`, a `money` fee). The scope is the rule, not only the picker: the
  invoke holds the id to it, so another client's sample is `NOT_FOUND` on
  `sample_id` before the handler runs — no `assert` repeats it.
- `analysis.start`, `.complete`, `.cancel` — pure transitions (one `read`).
- `analysis.release` — stamps `released_at` and **snapshots** the fee into
  `revenue`: a settled fact, not a live sum.

**Views and the app.**

| View | Block | Shows |
|---|---|---|
| `Lab.dashboard` | `composite` | five metrics (open, ready to release, overdue via `is_overdue`, open-fee sum, revenue this month vs last with a sparkline), a count-by-status chart, a grouped table (three measures per sample matrix), a "Due next" table |
| `Analysis.bench` | `queue` | every open analysis; each row offers only the next step its state allows |
| `Analysis.outgoing` | `queue` | complete analyses, oldest due date first, "Release" as the one action |
| `Analysis.board` | `board` | analyses by status, one next step per card, overdue from `is_overdue`, the five newest released analyses, a board-level "+ New" and one in the Queued column |
| `Analysis.list` | `table` | presets with counts, the legal next step per row, a "Total" footer over `fee` |
| `Analysis.record` | `detail` | header with status pill and money metrics, then the property grid (opened from a row; not in the nav) |
| `Client.directory` | `composite` | client list beside the selected client's record, its analyses and samples stacked below |

Things to notice: money needs no `format`; "Work by matrix" groups by a link
path (`sample.matrix`) in enum order; Release takes a row off the bench (Start
and Mark complete update it in place); `overdue` paints only queued and running cards.

**The scenario** seeds with `objects.import`, parents first — two clients,
their samples, and one analysis from last year that arrives already `released`
(a create could not place it). Then it drives the loop: a request the scope
refuses, a real request, the FSM guard, start → complete → release, the revenue
sum in DKK (history included), the derived values, one overdue analysis.

## Adapt it

Rename the nouns, keep the shape, and keep the same bar: `canon catalog
validate` with zero errors, `canon catalog audit` with no `C1-shadow-entity` or
`C10-float-money`, a scenario of your own loop under `canon catalog test`, and a
look at the rendered app.
