# Declaring custom ActionTypes — `actions.declare`

`actions.declare` is the agent's centerpiece — the platform Action that
registers a new, custom ActionType in the org's catalog from JSON. The
handler is the declarative DSL described in
[Phase 17.5 §2](../../../phases/17.5-declarative-handlers.md#2-the-dsl-is-itself-a-zod-schema--packagescanonsrchandler-dslschemats);
the surrounding ActionType shape (name, governance, idempotency,
fixture, transition marker) is the same one platform-tier Actions use.

Once an action is declared it shows up on every surface — REST, CLI,
MCP — without a restart. CLI per-action sugar (`canon
lead.convert --lead-id ld_…`) is generated automatically from the input
schema.

This page focuses on the **declaration envelope** — every field, what
it does, and how the runtime composes them. For copy-paste handler
bodies, see the [Handler DSL cookbook](handler-dsl-cookbook.md).

---

## The full envelope

```json
{
  "name": "lead.convert",
  "description": "Convert a qualified lead into a Customer + primary Contact.",
  "object_type": "lead",
  "input": {
    "lead_id": { "type": "string" }
  },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Lead.status",
    "from": ["qualified"],
    "to": "converted"
  },
  "handler": {
    "steps": [ /* see Handler DSL cookbook */ ],
    "returns": { /* optional */ }
  },
  "fixture": {
    "principal": { "kind": "agent", "id": "agt_…", "org_id": "org_…" },
    "args": { "lead_id": "ld_fixture" },
    "expectedEnvelope": { "data": { "lead_id": "ld_fixture", "customer_id": null, "contact_id": null } }
  }
}
```

Drop into `--input @lead-convert.json`:

```sh
canon actions invoke actions.declare --input @lead-convert.json
```

Or via the session.yaml form:

```yaml
action: actions.declare
input:
  name: lead.convert
  description: Convert a qualified lead into a Customer + primary Contact.
  object_type: lead
  input:
    lead_id: { type: string }
  governance:
    requires_approval: false
    allowed_principals: [user, agent, service_account]
  idempotency: none
  transition:
    state_machine: Lead.status
    from: [qualified]
    to: converted
  handler:
    steps:
      - op: read
        object: lead
        where: { id: $input.lead_id }
        as: lead
      # … more steps; see Handler DSL cookbook
    returns:
      lead_id: $input.lead_id
      customer_id: $customer.id
      contact_id: $contact.id
```

---

## Field-by-field

### `name`

Format: `<object>.<verb>` — lowercase, dot-separated. Regex:
`^[a-z][a-z0-9]*\.[a-z][a-z0-9_]*$`. Examples: `lead.convert`,
`invoice.issue`, `deal.send_proposal`.

Reserved meta-tool roots — the action name's `<object>` part may not
collide with these (Phase 17.6 §6 Open Q #5): `actions`, `objects`,
`sql_read`, `schema`, `skill`, `state-machines`, `auth`, `keys`,
`orgs`, `users`, `config`, `upgrade`.

### `description`

Required, non-empty. Surfaced in `canon actions list`,
`canon actions describe --json`, and the agent's `--explain` output.
Write it as if you were explaining the action to a future agent — one
crisp sentence stating the effect.

### `object_type`

Name of the primary ObjectType the action operates on. Used for
catalog grouping and surface generation. Must be a registered
ObjectType.

### `input`

JSON-Schema-shaped descriptor of the input schema. The platform
calls `buildSchemaFromJson(input)` to construct the runtime validator.
Supported field types: `string`, `number`, `boolean`. Optional fields:
`{ type: 'string', optional: true }`. Prefixed-id shortcut:
`{ type: 'string', prefix: 'ld_' }` — the runtime asserts the id starts
with the declared prefix.

### `governance`

```json
{
  "requires_approval": false,
  "allowed_principals": ["user", "agent", "service_account"]
}
```

- `requires_approval`: `true` enables the approval gate (Phase 22 work
  surfaces an approval queue); the envelope returns
  `approval_status: 'pending'` on first call.
- `allowed_principals`: at least one of `user`, `agent`,
  `service_account`. Calls from a principal kind not in the list
  return `FORBIDDEN`.
- `policy`: optional named policy reference for fine-grained
  authorisation (deferred to a later hardening phase).

### `idempotency`

One of:

- `none` — every call runs the handler; safe for transition actions
  where the FSM guard provides the natural retry barrier.
- `per-key` — when the caller supplies `idempotency_key`, the runtime
  caches the first envelope and replays it on retry. Use this for
  destructive operations the caller will retry on network failure.
- `natural-key` — the input shape itself is the idempotency key (e.g.,
  `email.send_template` keyed by `{template, to, customer_id}`); the
  runtime computes a stable hash and caches accordingly.

### `transition` (optional)

When set, declares "this is an FSM transition action". The runtime
auto-injects the guard, post-update, and on_enter/on_exit hooks — see
[`declaring-state-machines.md`](declaring-state-machines.md).

```json
{
  "state_machine": "Lead.status",
  "from": ["qualified"],
  "to": "converted"
}
```

- `state_machine` must reference a registered StateMachine (declaration
  order is flexible).
- `from`: either an array of source state names or the literal `"*"`
  (matches any non-terminal state).
- `to`: destination state name. Mutually exclusive with `to_expr`
  (computed-transition future feature).

The declaration is cross-validated against the StateMachine — drift
raises `VALIDATION/<drift_code>`. See
[Phase 18.5 §6](../../../phases/18.5-state-machines.md).

### `handler`

The DSL document — steps + optional returns block. Validated against
the effect/Schema in
[`packages/canon/src/handler-dsl/schema.ts`](../../../../packages/canon/src/handler-dsl/schema.ts).
See the [Handler DSL cookbook](handler-dsl-cookbook.md) for shapes.

### `fixture` (optional but strongly recommended)

The conformance-gate fixture. Without one, the action can't be
verified across surfaces (REST/CLI/MCP must produce byte-identical
envelopes for the same call). The fixture's `args` must satisfy the
declared input schema; `expectedEnvelope` is partial (the harness's
matchers handle `{ matches: '^cust_' }`, `length: 1`, etc.).

### `presentation` (optional)

The **human presentation block** — how a generated UI labels, groups, confirms,
and reports the action. It changes **nothing** about invocation: `name` stays the
only invocation id on every surface and in agent discovery, and the runtime
governance gate remains the sole authority over whether a call is allowed. This
block only shapes the *button*. Declared here on `actions.declare` (and accepted
identically by `actions.update`), it is projected onto the discovery wire
(`actions describe`/`list`) so any renderer can consume it.

Every field is **optional** and **purely additive** — omit the block and the UI
falls back to humanising the raw `name`. Excess/misspelled keys (`{ colour }`)
**fail decode**, so a typo is caught at declare, not silently dropped.

```json
{
  "presentation": {
    "label": "Advance deal",
    "description": "Move this deal to the next pipeline stage.",
    "group": "Lifecycle",
    "icon": "arrow-right",
    "emphasis": "primary",
    "destructive": false,
    "confirm": "Advance this deal to the next stage?",
    "success": "Deal advanced.",
    "pending": "Advance queued for approval.",
    "refresh": "refresh"
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `label` | non-empty string | Button/menu caption. Falls back to a humanised `name`. |
| `description` | non-empty string | One-line human blurb, distinct from the agent-facing `description`. |
| `group` | non-empty string | Section key that folds related actions into a shared menu/toolbar group (e.g. `"Lifecycle"`, `"Comms"`). |
| `icon` | non-empty string | Lucide icon name shown next to the label. |
| `emphasis` | `"primary"` \| `"secondary"` | Toolbar prominence. `primary` claims the inline CTA; `secondary` is subdued/ghost. |
| `destructive` | boolean | `true` styles the control as dangerous and — when `confirm` is present — gates the invoke behind a confirmation. Presentation only; not an authz gate. |
| `confirm` | non-empty string | Confirmation-prompt copy shown before a (typically destructive) invoke. |
| `success` | non-empty string | Success-toast copy after a completed (non-pending) invoke. |
| `pending` | non-empty string | Copy shown when the invoke enqueues an approval request. |
| `refresh` | `"refresh"` \| `"navigate"` \| `"none"` | Post-success hint: re-read the current view, navigate to the linked record, or leave it untouched. Omitted ⇒ renderer default (`refresh`). |

**`group` folds the toolbar; `emphasis: "primary"` claims the CTA.** In the
auto-generated record toolbar, actions sharing a `group` collapse into one
dropdown menu, and the single action marked `emphasis: "primary"` is lifted out
as the prominent inline button. This is exactly the fix the
`elegance_ungrouped_action_wall` advisory nudges toward when an object has more
custom actions than the toolbar's inline budget.

**Preserve-on-omit vs explicit-`{}` clearing.** `presentation` is a display
refinement, so an `actions.update` (or a re-`declare` reconcile) that **omits**
`presentation` entirely **preserves** the previously-stored block — a bundle file
that never mentions presentation won't silently strip it. To actually **remove**
a presentation block via a bundle, pass `"presentation": {}` **explicitly**;
supplying a non-empty `presentation` (even to change one field) replaces the whole
block with what you sent.

> **Interplay with ObjectType `display.actions` overrides.** An ObjectType (or a
> ViewSpec) can *override* how a specific action is presented via
> `display.actions` (see the `canonify-app-authoring` skill's *objecttype-refinement*).
> When both exist, the override's `label` and `emphasis` **win** over this block,
> and `hidden` removes the action from the surface entirely — but **`group` comes
> only from `presentation`** (an override never regroups an action), and
> `destructive`/`confirm`/`success`/etc. always pass through from here. So: put
> grouping and the confirm/toast copy on the *action*; use `display.actions` for
> per-object relabels, emphasis, and hiding.

---

## What `actions.declare` does

The handler runs five validations in order before persisting the
declaration to `_action_types`:

1. **Schema surface check.** `actions.declare`'s own input is itself
   schema-validated (`actionsDeclareInput`). Garbage in fails here with
   `VALIDATION`.
2. **`buildSchemaFromJson(input)`.** Builds the runtime input schema.
   Bad descriptors fail with `VALIDATION/bad_input_schema`.
3. **Static type-check on the handler DSL.** Walks every step,
   resolves every `$ref` against the live registry. Errors cite the
   offending step + field — `VALIDATION/typecheck_failed`.
4. **`direct_state_write` check.** If `transition` is set, walks every
   `update` step's `set` map and rejects writing the FSM-bound
   property — `VALIDATION/direct_state_write`.
5. **Cross-validate transition marker against the StateMachine.**
   Drift surfaces with the §6 drift code in
   `error.details.code` — `VALIDATION/<drift_code>`.

On success:

- The ActionType is registered on the live Registry (visible on every
  surface immediately).
- The full declaration JSON is persisted to `_action_types` keyed by
  `(org_id, name)`. Reboot re-registers from these rows.
- Envelope returns `{ data: { name, registered: true }, audit_event_id }`.

---

## Idempotency on `actions.declare` itself

`actions.declare` is `idempotency: 'per-key'`. Two retry shapes:

| Submission | Outcome |
|---|---|
| Same `name`, same JSON, with same `idempotency_key` | Cached envelope replay — free. |
| Same `name`, same JSON, no `idempotency_key` | Detected at handler entry (declaration JSON canonicalised + compared with existing row) — no-op success. |
| Same `name`, **different** JSON | `INVALID_STATE/declaration_changed` — use `actions.update` to change a registered action's declaration. |
| Same `name` in Registry but no `_action_types` row | `INVALID_STATE/duplicate_name` — raced declaration or table desync; investigate. |

This makes agent retry loops safe: re-emitting the same JSON is always
either a free replay or a no-op success; only a deliberate change
forces explicit intent (`actions.update`).

---

## Companion Actions

- `actions.list [--json]` — list every registered ActionType (platform
  + custom). The discovery surface.
- `actions.describe <name> [--json]` — return the declaration JSON for
  one action. Includes input schema, governance, idempotency,
  transition, fixture, and a `cli_example` field — a ready-to-paste
  `canon <name> --input '<fixture.args>'` invocation (derived from the
  declared fixture; `--input '{}'` when none). `--explain` on any
  per-action sugar command returns the same payload as a shortcut.
- `actions.update <declaration>` — replace an existing declaration.
  Same schema, same type-check, same persistence; differs only in that it
  accepts a changed declaration where `actions.declare` would reject
  with `INVALID_STATE/declaration_changed`.
- `actions.delete <name>` — remove a custom ActionType. Auto-CRUD
  defaults are not removable via this action — disable them via
  `catalog.refine_object_type`'s `defaults` field instead.

---

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| Input schema descriptor malformed | `VALIDATION/bad_input_schema` | Fix the `input` field; see Phase 17.5 §11 Open Q #2 for the supported subset. |
| Handler references unknown ObjectType | `VALIDATION/typecheck_failed` | Declare or refine the ObjectType first; see [`declaring-objecttypes.md`](declaring-objecttypes.md). |
| Handler references unknown step bind / property | `VALIDATION/typecheck_failed` | Read the cited step's error message; correct the ref. |
| Handler writes the FSM-bound state column | `VALIDATION/direct_state_write` | Remove the `set: { status: … }` entry — the runtime auto-writes it. |
| Transition marker drifts from StateMachine | `VALIDATION/<drift_code>` | Re-declare the StateMachine OR fix the action's `transition.from` / `transition.to`. |
| Same name, different JSON, no `idempotency_key` | `INVALID_STATE/declaration_changed` | Use `actions.update` to overwrite. |
| `_action_types` INSERT failed | `VALIDATION/persist_failed` | Storage-layer failure; retry. If persistent, escalate. |

See [`error-codes.md`](error-codes.md) for the full kind table.

---

## See also

- [Handler DSL cookbook](handler-dsl-cookbook.md) — copy-paste handler bodies
- [Declaring StateMachines](declaring-state-machines.md) — `transition` marker semantics
- [Declaring ObjectTypes](declaring-objecttypes.md) — the registry your handler steps reference
- [Phase 17.5 §5](../../../phases/17.5-declarative-handlers.md#5-the-agent-entry-point--actionsdeclare) — full spec
- [`packages/canon/src/actions/actions/declare.ts`](../../../../packages/canon/src/actions/actions/declare.ts) — the implementation
