---
title: Worked example — a complete, small app bundle you can apply
description: A whole Canonify app as one on-disk bundle — three tables, refined ObjectTypes, a state machine, five custom actions, seven views, an app, and a scenario that proves the business loop. Read it, run it with `canon catalog test`, copy it.
---

# Worked example — the Workshop app

One complete app as one bundle, next to this file in `worked-example/`. A bicycle
repair workshop: customers own bikes, bikes come in for jobs, a job moves through
a lifecycle until the bike is collected and paid. Copy the shape, rename the nouns.

After `canon skills install canonify-app-authoring` it is at
`.agents/skills/canonify-app-authoring/resources/worked-example/`. One file
without installing:
`canon skill --name canonify-app-authoring resource worked-example/canon/actions/job.collect.json`.

```
worked-example/
  canon/                          # catalog.export layout (no manifest.json, no policies/ — both optional)
    schema/                       # customer, bike, job
    object-types/                 # enums, links, money, derived properties, display
    state-machines/Job.status.json
    actions/                      # job.book, job.start, job.mark_ready, job.collect, job.cancel
    views/                        # dashboard, workbench + pickup queues, board, list, record, customer directory
    apps/workshop.json            # nav order, default view, a count badge
  scenarios/workshop-loop/session.yaml
```

## Run it

```sh
cd .agents/skills/canonify-app-authoring/resources/worked-example
canon catalog validate canon          # one warning: missing_part_of on bike (deliberate)
canon catalog test canon scenarios    # no org, nothing durable
```

Expect `"passed": true` and `applied` counts of 3 schema, 3 object types, 5
actions, 1 state machine, 7 views, 1 app. The `missing_part_of` warning is
advisory: `part_of` would make a bike a part of its customer, but a bike has its
own jobs and record page, so it stays first-class.

To try it in a real org, apply to a **fresh or sandbox org**
(`canon catalog apply canon --force`): apply prunes every managed artifact the
bundle does not mention. To add it to an existing app, copy its files into your
own exported bundle ([catalog-bundle.md](catalog-bundle.md)).

## What each part shows

**Tables.** Money is `INTEGER` minor units (`quote`, `revenue` in øre);
timestamps are ISO `TEXT`; state columns are `TEXT NOT NULL DEFAULT
'<initial>'`, narrowed to enums by the refinement; `revenue` is nullable until
collection.

**ObjectTypes.**
- Belongs-to links over every FK (`job.bike`, `job.customer`, `bike.customer`,
  each `fk_on: source`); the parent declares the `many` side with `on_delete`.
- `{ type: money, currency: DKK }` renders `DKK 1,250.00` from `125000`.
- Enums carry `valueLabels` and `valueColors` (`ready` is `good`, `cancelled`
  `hot`).
- Derived properties read like columns: `quote_incl_vat` (derived `money` with
  a currency and a `label`) and `is_overdue` (derived `boolean`, used by the
  board, a dashboard tile and filters).
- `display.title` (`{make} {model}` for a bike) makes every FK render as a
  title, not an id — table and board cells, record headers, queue rows.
- `job` sets `defaults: { create: false, delete: false }`: jobs are created only
  through `job.book` and cancelled, never deleted; history arrives by import.

**State machine.** `booked → in_progress → ready → collected`, plus `cancelled`
from any open state. A state left and re-entered (waiting for parts ⇄ in
progress) is a [two-way pair](../../canonify-objects-actions/resources/state-machines.md);
a job that starts past `booked` (checked in on arrival) is a
[create-then-transition](patterns.md).

**Actions.**
- `job.book` — a create with a rule: it reads the bike, `assert`s it belongs to
  the given customer, and copies the bike's `customer_id`. Its `quote` input is
  `money` with a currency. Its `presentation` builds the form: `formFields`
  (customer first), `inputDefaults` (priority `normal`), `inputFilters` (the bike
  picker scoped to the chosen customer). The filter is convenience; the
  `assert` is the rule.
- `job.start`, `job.mark_ready`, `job.cancel` — pure transitions (one `read`).
- `job.collect` — stamps `collected_at` and **snapshots** the quote into
  `revenue`: a settled fact, not a live sum.
- Every action has `presentation` copy; the top-level `description` is for
  agents and never shown to a person.

**Views and the app.**

| View | Block | Shows |
|---|---|---|
| `Workshop.dashboard` | `composite` | five metrics (open, ready, overdue via `is_overdue`, open-quote sum, revenue this month vs last with a sparkline), a count-by-status chart, a grouped table (three measures per bike type), a "Due next" table |
| `Job.workbench` | `queue` | every open job; each row offers only the next step its state allows |
| `Job.pickup` | `queue` | ready jobs, oldest promise first, "Hand over" as the one action |
| `Job.board` | `board` | jobs by status, one next step per card, overdue from `is_overdue`, the five newest collected jobs, a board-level "+ New" and one in the Booked column |
| `Job.list` | `table` | presets with counts, the legal next step per row, a "Total" footer over `quote` |
| `Job.record` | `detail` | header with status chip and money metrics, then the property grid (opened from a row; not in the nav) |
| `Customer.directory` | `composite` | customer list beside the selected customer's record, its jobs and bikes stacked below |

Things to notice: money needs no `format` (every aggregate carries its unit);
"Revenue this month" compares a running month against the same days of last
month; "Work by bike type" groups by a link path (`bike.kind`) in enum order;
transition actions appear only on rows in one of their `from` states, with no
`showWhen`; Start and Mark ready keep a row in the workbench (it updates in
place) while Hand over takes it out; the board's four filtered statuses are its
four columns, and `overdue` paints red only on booked and in-progress cards.

**The scenario** seeds with `objects.import`, parents first — two customers,
their bikes, and one job from last year that arrives already `collected` (a
create could not place it). Then it drives the loop: a booking the `assert`
refuses, a real booking, the FSM guard, start → ready → collect, the revenue sum
in DKK (history included), the derived values, one overdue job.

## Adapt it

Rename the nouns, keep the shape, and keep the same bar: `canon catalog
validate` with zero errors, `canon catalog audit` with no `C1-shadow-entity` or
`C10-float-money`, a scenario of your own loop under `canon catalog test`, and a
look at the rendered app.
