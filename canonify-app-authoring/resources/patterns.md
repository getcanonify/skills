---
title: App patterns — human codes and create-then-transition
description: Two patterns apps keep needing that no single verb names — minting a human-readable code like J-1042, and a create action whose new record starts past the state machine's initial state (a 'Check in a bike' on the parent record, a '+ New' in a board column).
---

# App patterns

Both patterns use only the handler DSL
([actions-handler-patterns.md](../../canonify-objects-actions/resources/actions-handler-patterns.md))
and state machines ([state-machines.md](../../canonify-objects-actions/resources/state-machines.md)).

## Minting a human code (`J-1042`)

A handler cannot concatenate strings; a derived property can. Store the sequence
as an integer column, mint the next number in the create with `read_aggregate`
`max` + 1, and expose the code as a derived property:

```json
{
  "steps": [
    { "op": "read_aggregate", "object": "job", "where": {}, "max": "number", "as": "last" },
    { "op": "insert", "object": "job", "as": "job", "values": {
      "number": { "op": "+", "left": "$last.max", "right": 1 },
      "title": "$input.title" } }
  ]
}
```

On the ObjectType: `"code": { "kind": "derived", "type": "string", "label":
"Job", "expression": "'J-' || job.number" }`.

- Declare the column `INTEGER UNIQUE` in the DDL, so a collision is refused,
  not duplicated.
- `where: {}` matches every row. `max` over an empty type is `null`, and
  declare does NOT flag `$last.max + 1` — the action is refused at RUN time
  (`VALIDATION`: `operator '+' requires both operands to be numbers`) the first
  time it mints on an empty type. Seed the first row (import it with its
  `number`) before the action runs.

## Create-then-transition — a new record that starts past the initial state

A create cannot write a state-machine status (`direct_state_write`), so a plain
create always lands in the machine's `initial` state. When a record must start
further along — a bike checked in on arrival, not just booked — insert it and
name it with a `transition` step in the same action, under a `transition` marker
whose `from` is the initial state:

```json
{
  "name": "job.check_in",
  "description": "Check a customer's bike in now: create the job and put it straight on the bench.",
  "object_type": "job",
  "input": { "customer_id": { "type": "string", "prefix": "cus_" }, "bike_id": { "type": "string", "prefix": "bik_" }, "title": { "type": "string", "min": 1 } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent"] },
  "idempotency": "per-key",
  "transition": { "state_machine": "Job.status", "from": ["booked"], "to": "in_progress" },
  "subject": { "param": "customer_id", "object_type": "customer" },
  "handler": {
    "steps": [
      { "op": "insert", "object": "job", "as": "job", "values": { "customer_id": "$input.customer_id", "bike_id": "$input.bike_id", "title": "$input.title" } },
      { "op": "transition", "object": "job", "where": { "id": "$job.id" } }
    ],
    "returns": { "job_id": "$job.id" }
  }
}
```

- The transition must be listed in the state machine (`via: job.check_in`,
  `from: [booked]`, `to: in_progress`) like any other.
- With `subject` naming the parent (`customer_id` → `customer`), the action is
  offered on the customer record (never on a job record) with the customer
  pre-bound: "Check in a bike" opens a form asking only for the bike and the
  title. When the customer page lists its jobs, the offer sits in that jobs
  table's header (it creates a job), not on the record toolbar — once per
  page.
- As a board's `createAction` (or a `lanes.in_progress.create`) it puts a
  "+ New" in the `in_progress` column; a plain create (`job.book`) puts one in
  the `initial` state's column (`lanes.booked.create`).
