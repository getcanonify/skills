---
title: Chapter 3 — Bind a StateMachine
description: Use state_machines.register to bind a finite-state machine to an enum property. Transitions name the Action that drives them; on_enter/on_exit hooks enqueue follow-up actions. Registering the FSM is what lets your transition actions get a free guard plus auto state-write.
---

# Chapter 3 — Bind a StateMachine (`state_machines.register`)

A StateMachine binds to an ObjectType **enum property** (narrowed in
Chapter 2) and declares the legal transitions between its values.
Registering the machine is what gives the transition actions you declare in
Chapter 4 their power: a free pre-handler guard, a free post-handler state
write, and free `on_enter` / `on_exit` follow-up hooks.

> `Deal.stage` below is an illustrative lifecycle, not a product commitment.

## The action

| | |
|---|---|
| **Action** | `state_machines.register` |
| **Input** | `{ kind: 'state_machine', name, objectType, property, initial, states, transitions, on_enter?, on_exit? }` |
| **Envelope** | `{ data: { name, registered: true }, audit_event_id, approval_status: 'not_required' }` |

- `name` — the machine's id, conventionally `<ObjectType>.<property>`
  (e.g. `Deal.stage`). This is the string a transition action's
  `transition.state_machine` marker references.
- `objectType` / `property` — what it binds to. `property` must be an
  `enum` on that ObjectType.
- `initial` — the value new rows start in.
- `states` — each enum value, with a `label` and optional `terminal: true`.
- `transitions` — each carries `via` (the Action name that drives it),
  `from` (a list of source states, or `"*"` for any non-terminal state —
  `"*"` also onboards a row whose state column is still NULL; an explicit
  list never matches NULL),
  and `to` (the destination).

## The example machine

```yaml
action: state_machines.register
input:
  kind: state_machine
  name: Deal.stage
  objectType: deal
  property: stage
  initial: open
  states:
    open:           { label: Open }
    proposal_sent:  { label: Proposal sent }
    won:            { label: Won, terminal: true }
    lost:           { label: Lost, terminal: true }
  transitions:
    - { via: deal.send_proposal, from: [open],                to: proposal_sent }
    - { via: deal.win,           from: [proposal_sent],       to: won }
    - { via: deal.lose,          from: [open, proposal_sent], to: lost }
  on_enter:
    won:
      - action: email.send_template
        inputs:
          template: deal_won
          to:       $row.customer_id
```

You can register the machine **before or after** the transition actions it
names — declaration order is flexible; the cross-validation happens when
both sides exist.

## What registration buys your transition actions

Once `Deal.stage` is registered, any action whose declaration carries a
matching `transition` marker (Chapter 4) gets three slabs of machinery
wrapped around its handler **for free**:

1. **Pre-handler guard.** Rejects with `INVALID_STATE` if the row's
   current `stage` is not in the transition's `from`.
2. **Post-handler state write.** The runtime auto-writes
   `UPDATE deal SET stage=<to> WHERE id=…` after the handler returns. Your
   handler must **not** write the state column itself — doing so is
   rejected at `actions.declare` with `VALIDATION/direct_state_write`.
3. **Hook enqueue.** `on_enter[<to>]` / `on_exit[<from>]` hooks resolve
   against the post-transition row and enqueue the named follow-up actions
   into the outbox, committed atomically with the transition. The `$row.*`
   refs in hook `inputs` are passthrough — they resolve at execute time
   against the actual row, not against session scope.

## Invoke it — three surfaces

**CLI:**

```sh
canon actions invoke state_machines.register --input @deal-stage.json
```

**REST** (`{ input }` wraps the `state_machine` body):

```sh
curl -X POST https://api.canonify.app/v1/actions/state_machines.register \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "kind": "state_machine",
      "name": "Deal.stage",
      "objectType": "deal",
      "property": "stage",
      "initial": "open",
      "states": {
        "open": { "label": "Open" },
        "proposal_sent": { "label": "Proposal sent" },
        "won": { "label": "Won", "terminal": true },
        "lost": { "label": "Lost", "terminal": true }
      },
      "transitions": [
        { "via": "deal.send_proposal", "from": ["open"], "to": "proposal_sent" },
        { "via": "deal.win", "from": ["proposal_sent"], "to": "won" },
        { "via": "deal.lose", "from": ["open", "proposal_sent"], "to": "lost" }
      ]
    }
  }'
```

**MCP** — `actions.invoke` with `name: "state_machines.register"` and the
body under `args`.

## Inspect what you registered

```sh
canon state-machines list --json
canon state-machines render Deal.stage   # ASCII state diagram
```

The render is the fastest way to confirm the graph matches your intent
before you declare the transition actions.

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| `property` isn't an enum | `VALIDATION` | narrow it first in Chapter 2 |
| A transition references an unknown state | `VALIDATION` | every `from` / `to` must be a declared state value |
| Transition action's marker drifts from the machine | `VALIDATION/<drift_code>` (at `actions.declare`) | align the action's `transition.from` / `transition.to` with the machine, or re-register the machine |

## Next

→ [Chapter 4 — Declare a custom ActionType](actions-multi-step.md): declare
`deal.win`, `deal.send_proposal`, `deal.lose` and wire them to this machine.
