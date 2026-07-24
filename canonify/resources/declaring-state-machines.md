# Declaring StateMachines — `state_machines.register`

State machines bind to an ObjectType's enum property and enforce
"which transitions are legal from which state, and which Action drives
each transition". They are the Phase 18.5 contract; this Action surface
lets agents declare them through CLI/REST/MCP without writing TS.

Read [Phase 18.5](../../../phases/18.5-state-machines.md) for the full
FSM contract — what gets auto-injected (guard, post-update, on_enter
hooks), the cross-validation rules, and the runtime semantics. This
page covers only the **declaration** shape an agent emits.

Prerequisite: the target property must already be a typed `enum`. If
the underlying column is still a permissive `string`, narrow it first
via [`catalog.refine_object_type`](declaring-objecttypes.md).

---

## Declaration shape

```yaml
action: state_machines.register
input:
  kind: state_machine
  track: customer                  # 'platform' or 'customer'
  name: Lead.status                # <ObjectType>.<property> by convention
  objectType: lead
  property: status
  initial: new
  states:
    new:          { label: New }
    contacted:    { label: Contacted }
    qualified:    { label: Qualified }
    converted:    { label: Converted,    terminal: true }
    disqualified: { label: Disqualified, terminal: true }
  transitions:
    - { via: lead.contact,    from: [new],                       to: contacted }
    - { via: lead.qualify,    from: [contacted],                 to: qualified }
    - { via: lead.disqualify, from: [new, contacted, qualified], to: disqualified }
    - { via: lead.convert,    from: [qualified],                 to: converted }
  on_enter:
    converted:
      - action: email.send_template
        inputs:
          template:    welcome
          to:          $row.email
          customer_id: $row.customer_id
```

JSON equivalent (drop into `--input @file.json`):

```json
{
  "kind": "state_machine",
  "track": "customer",
  "name": "Lead.status",
  "objectType": "lead",
  "property": "status",
  "initial": "new",
  "states": {
    "new":          { "label": "New" },
    "contacted":    { "label": "Contacted" },
    "qualified":    { "label": "Qualified" },
    "converted":    { "label": "Converted",    "terminal": true },
    "disqualified": { "label": "Disqualified", "terminal": true }
  },
  "transitions": [
    { "via": "lead.contact",    "from": ["new"],                       "to": "contacted" },
    { "via": "lead.qualify",    "from": ["contacted"],                 "to": "qualified" },
    { "via": "lead.disqualify", "from": ["new", "contacted", "qualified"], "to": "disqualified" },
    { "via": "lead.convert",    "from": ["qualified"],                 "to": "converted" }
  ],
  "on_enter": {
    "converted": [
      {
        "action": "email.send_template",
        "inputs": {
          "template":    "welcome",
          "to":          "$row.email",
          "customer_id": "$row.customer_id"
        }
      }
    ]
  }
}
```

---

## Field semantics

| Field | Meaning |
|---|---|
| `track` | `platform` (Canonify engineers, fixed-shape) or `customer` (agent-declared, lifecycle-managed). Agents declaring a SM in their org always use `customer`. |
| `name` | Stable identifier — referenced from `actions.declare`'s `transition.state_machine` field. Convention: `<ObjectType>.<property>` capitalised (e.g., `Lead.status`, `Invoice.status`). |
| `objectType` | The ObjectType whose property the FSM governs. Must be a registered ObjectType. |
| `property` | The property name on the ObjectType. Must be `kind: mapped`, `type: enum`. |
| `initial` | Default state set on row creation when the column is unspecified. Must be one of the `states` keys. |
| `states` | Map of state name → metadata. `label` is required; `terminal: true` marks states from which no non-terminal transition is allowed. `description`, `color`, `icon` are surfaced in renders. |
| `transitions` | The legal moves. `via` is the Action name that drives the transition (declared via `actions.declare`, or auto-CRUD-generated). `from` is a list of source states, or the literal `"*"`. `"*"` matches any non-terminal state **and also onboards a never-initialized row** — one whose state column is still NULL (created before the SM governed the property, or written by an importer). An explicit `from: [...]` list never matches a NULL state (you can't leave a state you're not in), so a wildcard transition is the only way to move an uninitialized row into the machine. `to` is the destination state. |
| `on_enter[<state>]` / `on_exit[<state>]` | Hooks: each is a list of `{ action, inputs }` records that the runtime auto-invokes after the state transition lands. The `inputs.*` map uses the `$row.<col>` / `$row_before.<col>` ref grammar — these resolve against the post-transition (and pre-transition) row inside the same tx; they are **passthrough** refs that don't leak into the session scope. |

> **`states.<name>.color` — the same five tokens.** A state's `color` feeds the
> same per-value color channel as an enum property's `valueColors`, so it colors
> the status chip / board column. It accepts the five white-label tokens
> (`hot | warm | cool | info | neutral`; the full table lives in
> [declaring-objecttypes › *Enum value colors*](declaring-objecttypes.md)). As a
> convenience, common color **names** normalize to a token — `red`/`rose`/`danger`
> → `hot`, `amber`/`orange`/`yellow` → `warm`, `green`/`emerald`/`teal` → `cool`,
> `blue`/`sky`/`indigo`/`purple` → `info`, `gray`/`slate`/`muted` → `neutral`.
> **Gotcha: a raw hex (`#e11d48`) is silently dropped** — the state falls back to
> the default neutral / hashed treatment rather than rendering that hex. Author a
> token (or a mappable color name), never a hex.

---

## What the runtime auto-injects

Once `state_machines.register` succeeds, every transition Action whose
declaration carries the matching `transition: { state_machine: …,
from, to }` marker gets three slabs of runtime machinery wrapped around
its handler **for free**:

1. **Pre-handler guard.** Reads the row's current state; rejects with
   `INVALID_STATE` if it's not in `transition.from`. Envelope error
   `details` includes `{ current_state, allowed_from }` so the agent
   can recover. For a never-initialized (NULL) row,
   `details.current_state` is `null` and the message reports
   `<uninitialized>`; such a row is accepted only by a `from: "*"`
   transition.
2. **Post-handler state write.** After the handler returns, the
   runtime runs `UPDATE <table> SET <property> = '<to>' WHERE id = …`
   inside the same tx. The handler must **not** write the state
   column itself — `actions.declare` rejects with
   `VALIDATION/direct_state_write` if you try.
3. **Hook enqueue.** `on_enter[<to>]` and `on_exit[<from>]` hooks are
   resolved against the post-transition row and enqueued into the
   `_outbox` for follow-up dispatch.

This means a transition Action's declarative handler is **just the
domain-side effect** — stamp a `contacted_at` timestamp, materialise a
Customer + Contact, etc. The state flip happens for free.

---

## Cross-validation with `actions.declare`

The Registry cross-validates state machines against any matching
declared Actions:

- An Action's `transition.state_machine` must point at a registered SM
  (declaration order is flexible — either way around works; the
  cross-validator runs again on the second registration).
- An Action's `transition.from` must be a subset of the union of all
  `from` lists for the SM's transitions where `via` matches the
  Action's name. The SM is the authoritative source.
- An Action's `transition.to` must equal the SM's `to` for the
  matching `via`.
- If the source state is `terminal: true`, the action cannot transition
  away from it — terminal states are sinks.
- If the Action carries a transition marker but the underlying
  property isn't an enum on the ObjectType, drift is reported.

Failures surface as `VALIDATION/<drift_code>` from `actions.declare`
or `state_machines.register`, depending on which side committed second.
See [Phase 18.5 §6](../../../phases/18.5-state-machines.md) for the
full validation matrix.

---

## Discovering registered StateMachines

```sh
canon state-machines list --json
canon state-machines describe Lead.status --json
canon state-machines render Lead.status        # ASCII diagram
```

`render` emits a state diagram an agent or human can read at a glance.

---

## See also

- [Declaring custom ActionTypes](declaring-actions.md) — Actions reference SMs via the `transition` marker
- [Phase 18.5](../../../phases/18.5-state-machines.md) — FSM contract, guard semantics, hook execution model
- [Phase 17.5 §8](../../../phases/17.5-declarative-handlers.md#8-statemachine-declaration--state_machinesregister) — declarability spec
- [`tests/scenarios/minimal-crm/session.yaml`](../../../../tests/scenarios/minimal-crm/session.yaml) — `Lead.status` and `Deal.stage` declared in context
