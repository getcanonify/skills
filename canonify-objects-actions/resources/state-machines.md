# State machines — `state_machines.register`

A StateMachine binds to one `enum` property of an ObjectType (narrow the column
first — [object-property-types.md](object-property-types.md)) and declares which
transitions are legal and which action drives each.

```yaml
action: state_machines.register
input:
  kind: state_machine
  track: customer
  name: Lead.status
  objectType: lead
  property: status
  initial: new
  states:
    new:          { label: New, meaning: live }
    contacted:    { label: Contacted, meaning: live }
    qualified:    { label: Qualified, meaning: live }
    converted:    { label: Converted, meaning: terminal-good }
    disqualified: { label: Disqualified, meaning: terminal-bad }
  transitions:
    - { via: lead.contact,    from: [new],                       to: contacted }
    - { via: lead.qualify,    from: [contacted],                 to: qualified }
    - { via: lead.disqualify, from: [new, contacted, qualified], to: disqualified }
    - { via: lead.convert,    from: [qualified],                 to: converted,
        stamps: { at: converted_at, by: converted_by } }
  on_enter:
    converted:
      - action: email.send_template
        inputs: { template: welcome, to: $row.email, customer_id: $row.customer_id }
```

Envelope: `{ data: { name, registered: true }, audit_event_id }`.

| Field | Meaning |
|---|---|
| `track` | Always `customer` for a machine you declare (`platform` is the platform's own). |
| `name` | The id a transition action's `transition.state_machine` names. Convention `<ObjectType>.<property>`. |
| `objectType` / `property` | What it binds; the property must be `kind: mapped`, `type: enum`. |
| `initial` | The state a new row starts in. Must be a `states` key. |
| `states` | State → `{ label, meaning?, terminal?, description?, color?, icon? }`. `meaning` says what the state is ([below](#state-meanings)); a `terminal` state — or one meaning `terminal-good` / `terminal-bad` — has no way out and settles the record ([below](#a-terminal-state-settles-the-record)). |
| `transitions` | `{ via, from, to, stamps? }`: `via` is the driving action, `from` a list of states or `"*"`, `to` the destination. `"*"` matches any non-terminal state **and** onboards a row whose state is still NULL; an explicit list never matches NULL. `stamps` records who moved the record and when ([below](#transition-stamps)). |
| `on_enter[<state>]` / `on_exit[<state>]` | `[{ action, inputs }]` follow-ups. `inputs` use `$row.<col>` / `$row_before.<col>`, resolved against the post- / pre-transition row at execute time (passthrough refs, not session scope). |
| `unmanaged_states` | Legacy values that predate the machine. |

A state's `color` feeds the same channel as an enum's `valueColors` (the row's
pill, board column). Use a token from the
[`valueColors` table](object-property-types.md); common names map onto one —
`red` / `rose` / `danger` → `hot`, `amber` / `orange` / `yellow` → `warm`,
`green` / `emerald` / `teal` / `lime` / `mint` → `good`, `slate` / `steel` →
`cool`, `blue` / `sky` / `indigo` / `purple` → `info`, `gray` / `stone` /
`muted` → `neutral`. A hex (`#e11d48`) is silently dropped.

**A two-way pair** (running ⇄ on hold) is two actions, one per direction,
each offered only in its own `from` state.

### State meanings

Each state may declare what it means; the stepper, lanes and state hues read
it, never the state's colour:

| `meaning` | The state is | Exits |
|---|---|---|
| `live` | work in progress — a record moves on from it | must have one; not `terminal: true` |
| `terminal-good` | the end that succeeded (paid, delivered, attended) | none — it is terminal |
| `terminal-bad` | an end that failed (lost, no-show, declined) | none — it is terminal |
| `sink` | where a record leaves the work (cancelled, discarded, superseded) | any; mark `terminal: true` when it is an end |

A record header's lifecycle stepper runs from `initial` to the
**terminal-good** state, through no sink; a terminal-bad or sink state is a
branch, never the goal. A machine with no terminal-good state draws no
stepper. A state rejected but resubmittable is `live`, not `terminal-bad`.
Refused at register (`bad_meaning`): an unknown meaning (the refusal lists the
four), a terminal meaning on a state a transition leaves, `live` on a
`terminal: true` state. The governed enum declares no `valueMeanings` of its
own: the wire's `valueMeanings` for it are its states' meanings.

### Transition stamps

`stamps: { at: <timestamp property>, by: <principal property> }` on a
transition writes, with the state and in the same transaction, the org
clock's now (an owner's `demo_now` pin included) into `at` and the invoking
principal's id into `by`. Either may be omitted. `at` must be a mapped
`timestamp`, `by` a mapped `string` declared `principal: true` (reads then
name the actor); anything else is refused at register (`bad_stamps`). The
action's `canon actions describe` lists them under `transition.stamps`, and
the schema wire under the enum's `stateMachine.stamps`.

### To-cases

Confirm → `confirmed` or `on_hold` is one action whose `to` is a case list,
`[{ "when": [{ "field": "over_limit", "op": "eq", "value": true }], "to": "on_hold" }, { "to": "confirmed" }]`:
after the handler the row lands in the first case whose `when`
([conditions](actions-handler-patterns.md#conditions--one-grammar-for-every-surface)
on its mapped or derived properties; a `$principal.*` value is the invoker)
holds, else in the last, `when`-less default (none is `transition_to_no_default`). The machine lists a transition
`via` the action to each target; a board never fires one by drop.

**Order.** Declare the actions first: `state_machines.register` refuses a
transition `via` or hook `action` that is not declared yet (`unknown_action`,
`unknown_hook_action`), while an action whose marker names a machine not yet
registered is accepted and checked when the machine registers. `canon catalog
apply` orders a bundle for you.

## What a transition action gets for free

An action whose declaration carries `transition: { state_machine, from, to }`
(see [actions-input-governance.md](actions-input-governance.md)) is wrapped in:

1. **Guard** — `INVALID_STATE` unless the row's state is in `from`. `details`
   carries `{ current_state, allowed_from, object_type, id }` to a caller who may
   read the state column (only `{ object_type, id }` otherwise). A NULL row
   reports `current_state: null` / `<uninitialized>`.
2. **State write** — after the handler returns, the runtime writes `to` (or
   the to-case that holds) in the same transaction. The handler never writes the bound property.
3. **Hooks** — `on_exit[<from>]` and `on_enter[<to>]` resolve against the row
   and enqueue to the outbox, committed with the transition.

So a transition handler is only the domain effect (stamp `contacted_at`,
create the customer). By default it moves ONE row, the action's subject
(`<type>_id`, `id`, or a declared or reference-inferred `subject` —
[actions-input-governance.md](actions-input-governance.md#subject)); without
one, registration fails with `no_id_input`.

### Moving a set in one call

Drop the id input and name the rows with `transition` steps, usually in a
`for_each`; the marker is unchanged:

```json
{
  "name": "order.lock_day",
  "description": "Lock every open order of a delivery day.",
  "object_type": "order",
  "input": { "location_id": { "type": "string" }, "delivery_date": { "type": "string" } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent"] },
  "idempotency": "none",
  "transition": { "state_machine": "Order.status", "from": ["open"], "to": "locked" },
  "handler": {
    "steps": [
      { "op": "read_many", "object": "order",
        "where": { "location_id": "$input.location_id", "delivery_date": "$input.delivery_date" },
        "as": "todays" },
      { "op": "for_each", "in": "$todays", "as": "o", "steps": [
        { "op": "transition", "object": "order", "where": { "id": "$o.id" },
          "when": { "ref": "$o.status", "in": ["open"] } }
      ] }
    ],
    "returns": { "candidates": "$todays.count" }
  }
}
```

- A `transition` step writes nothing itself; it names one row of the action's
  own ObjectType (`NOT_FOUND` if `where` matches none). After the handler, the
  runtime guards **every** named row, writes `to` and fires hooks per row, in
  the same transaction.
- One named row outside `from` fails the whole call with `INVALID_STATE`
  (details name the row) — decide membership with the step's `when`.
- Naming no row succeeds and moves nothing; naming a row twice moves it once.
- Such an action cannot back an `automatic` transition (`bad_schedule`).
- A `transition` step without a marker, or naming another ObjectType than the
  action's `object_type`, is `VALIDATION/invalid_transition_step`; the message
  names the `object_type` and `subject` to declare.
- **Moving rows from another type's page.** The step names rows of the
  action's `object_type`; the button's record is the `subject`, which may be
  another type. Add `"subject": { "param": "location_id", "object_type":
  "location" }` to `order.lock_day` and it is offered on a location's page,
  moving that location's orders.

## The bound property belongs to the machine

A new row starts in `initial`, and after that only a declared transition writes
the property. Every other writer is refused on every transport with
`VALIDATION`, `details.code: direct_state_write` (the message names the property
and the machine):

| Path | Result |
|---|---|
| Generated `<type>.create` (and a part created inside its root's create) | An omitted state becomes `initial`; `initial` passed explicitly is fine; any other state is refused and the message points at `objects.import`. |
| `objects.import` | Loads rows in **any** declared state; no guard, hook or reaction runs ([tables-and-import.md](tables-and-import.md)). |
| A declared handler whose `update` sets the property | Refused at declare time, with or without a marker. |
| Generated `<type>.update` / `<root>.replace` naming the property (on the root or on a part with its own machine) | Refused at invoke time; nothing in the call is written. Omit the property to update the rest. |
| A transition action | The one sanctioned path. |

The check runs at **invoke** time, so registering a machine on a live org
protects the column at once. The rule is "names the property", not "changes
it": a read-modify-write must strip the state column first.

To unlock, cancel or reset, **declare a transition** —
`{ via: order.unlock, from: [locked], to: open }` (often `requires_approval`),
`{ via: order.cancel, from: "*", to: cancelled }`. Rows that must arrive in a
later state (last year's released analyses) are imported, never created and walked
through each transition (that would fire every hook as if the work happened
today). A column another system keeps overwriting should stay a plain enum —
a machine is a promise that the column is transition-only.

### A terminal state settles the record

A record in a `terminal` state is settled: the generated `<type>.update`,
`<root>.replace` and `<type>.delete` refuse it on every transport with
`CONFLICT`, `details.reason: "TERMINAL_STATE"` (`{ object_type, id,
state_machine, property, terminal_states }`); nothing is written. Discovery
marks those verbs `terminal_locked: true` and lists the bound property's
`stateMachine.terminal` states, so the web offers no Edit or Delete on a
released analysis. A declared action is not locked: a correction on a settled
record is a deliberate, governed verb you declare under its own name
(`analysis.correct`, usually `requires_approval`). Reusing the generated name
instead (a hand-declared `analysis.update`) is refused `INVALID_STATE`
`auto_crud_collision` until you disable that verb first with
`catalog.refine_object_type` `defaults: { update: false }`.

Owned children are settled with their owner: a child whose parent's downward
link to it is `part_of` or `on_delete: cascade` (a released analysis's
`charge_line` rows) has its generated create (under that parent), update, replace and delete
refused with the same `CONFLICT` `TERMINAL_STATE`, `details.owner` naming
`{ object_type, id, link }`; declared actions (`analysis.add_charge`) stay unlocked,
and discovery marks the owning link `owning: true`. The generated temporal
`<type>.set` / `<type>.end` are locked the same way. An activity child (a note,
a comment, a message) opts out on the parent's link with `settles: false`: it
stays owned (cascade, nesting) but is still written under a settled parent, and
discovery serves the link `settles: false` (only an owning link takes it —
`settles_shape_invalid` otherwise).

## Cross-validation

- `transition.state_machine` must name a registered machine (checked when it
  registers, if the action came first).
- The action's `from` must be a subset of the `from`s of the machine's
  transitions whose `via` is that action, and every state its `to` can land in
  must be one of theirs.
- A `terminal` source cannot transition away.
- A marker on a property that is not an enum is drift.

Drift is `VALIDATION/<drift_code>` from whichever of `actions.declare` /
`state_machines.register` commits second.

**Re-registering** an existing `name` needs `expectedHash` (the
`declaration_hash` from `catalog.export`'s machine record) or `force: true`;
omitting both is `CONFLICT/HASH_REQUIRED`, a stale hash
`CONFLICT/STALE_DECLARATION` (drift names keys such as
`['transitions','on_enter']`), both together
`VALIDATION/CONTRADICTORY_DECLARATION_INTENT`. A first registration is never
guarded.

## Inspect

```sh
canon state-machines list --json
canon state-machines describe Lead.status --json
canon state-machines render Lead.status    # ASCII diagram
```
