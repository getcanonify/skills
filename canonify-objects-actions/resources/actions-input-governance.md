# Action input & governance

`actions.declare` registers a custom ActionType. The handler DSL (the typed
CRUD steps) is the *body*; everything around it — name, input descriptor,
governance, idempotency, transition marker — is the *envelope* the runtime
wraps the handler in. This page documents that envelope, verified against the
`actionsDeclareInput` schema in
`packages/canon/src/actions/actions/declare.ts`, the `ZodSchemaJson` /
`ZodFieldJson` input descriptor in
`packages/canon/src/handler-dsl/schema-from-json.ts`, and
`ActionGovernanceSchema` in `packages/canon/src/types.ts`.

## The declaration shape

`actions.declare`'s input (`actionsDeclareInput`) is a struct with
`onExcessProperty: 'error'` — unknown top-level fields fail the decode. The
fields:

| field | required | notes |
|---|---|---|
| `name` | yes | must match `^[a-z][a-z0-9_]*\.[a-z][a-z0-9_]*$` — i.e. `<object>.<verb>` |
| `description` | yes | non-empty |
| `object_type` | yes | the ObjectType this action is filed under |
| `input` | yes | the JSON input descriptor (below) |
| `governance` | yes | the governance block (below) |
| `idempotency` | yes | `none` \| `per-key` \| `natural-key` |
| `transition` | no | Phase 18.5 transition marker (below) |
| `handler` | yes | the handler DSL — see [actions-handler-patterns.md](actions-handler-patterns.md) |
| `fixture` | no | a conformance fixture `{ principal, args, expectedEnvelope? }` |

## The input descriptor

`input` is a flat record of field-name → descriptor (`ZodSchemaJson` /
`ZodFieldJson`). Each descriptor has:

- `type` — one of `string` \| `number` \| `boolean` \| `json` (note: **no
  `timestamp`/`enum`/`file` at the input layer** — those are property types,
  not input types; an enum input is expressed with `enum: [...]` over a
  `string`).
- `prefix?` — id-prefix hint (e.g. `cust`).
- `min?` / `max?` — numeric bounds.
- `enum?` — narrows a `string` to a literal union.
- `format?` — `email` \| `iso8601`.
- `optional?` — makes the field optional.
- `nullable?` — allows null.

```json
{
  "lead_id": { "type": "string" },
  "channel": { "type": "string", "enum": ["email", "phone", "in_person"] },
  "note": { "type": "string", "optional": true }
}
```

A bad input at invoke time produces a `VALIDATION` envelope with the issue
array in `error.issues`.

## Governance

`governance` (`ActionGovernanceSchema`) is a struct:

| field | required | meaning |
|---|---|---|
| `requires_approval` | yes | boolean — gate every call behind approval |
| `allowed_principals` | yes | non-empty array of `user` \| `agent` \| `service_account` |
| `policy` | no | a fine-grained policy reference resolved by the governance layer |
| `requires_approval_when` | no | a where-DSL predicate over the parsed input; when it evaluates true, *that call* is treated as `requires_approval: true` |
| `operation` | no | overrides which **policy operation** the invoke-authorization gate checks for this action (below) |

`requires_approval_when` reuses the Phase 20 where-DSL (`and`/`or`/`cmp`/`in`/
`is_null`); identifiers in field position are top-level input keys. Example:
`"amount_cents > 100000"` — calls over $1000 (in cents) need approval; smaller
ones don't. A denied-pending call returns the same
`approval_status: 'pending'` envelope as a static `requires_approval: true`.

`allowed_principals` is a literal allowlist of `user | agent | service_account`
**only** (`ActionGovernanceSchema` rejects anything else at declare time). The
other principal kinds in the platform — `app`, `external`, `public`, and `system`
— are **server-minted**, not values you list here. They reach a declared action
not by being named in `allowed_principals` but through dedicated policy branches
(an `app`/`external`/`public` submit is exempt from this per-action allowlist and
resolved by its manifest/form policy). The newest of these, **`system`**, is the
internal cron-scheduler identity (`SystemPrincipal`, `kind: 'system'`,
`purpose: 'cron'`) that the cron driver mints to run schedule targets and
reaction `via` actions on the platform's behalf. It is un-forgeable: `parsePrincipal`
rejects `kind: 'system'` outright, so no `X-Principal-Json` header,
`--principal-json` flag, or `CANONIFY_PRINCIPAL_JSON` env can claim it — the only
path to a `system` principal is direct internal construction inside the cron tick.

### `operation` — which policy operation an action gates on

When the runtime authorizes an invoke, it asks the policy layer to `decide`
against a **policy operation** for `(principal, action)`. By default that
operation is *derived from the action name*: `<type>.create` → `create`,
`<type>.update` → `update`, `<type>.delete` → `delete`, and everything else
(custom verbs like `deal.advance`) → `invoke`. `governance.operation` overrides
that derived default — the gate checks the operation you declare instead.

The legal values are the runtime's policy operations (`OperationSchema` in
`packages/canon/src/policy/access-policy.ts`):

```
get | list | navigate | aggregate | create | update | delete | invoke | manage
```

`get` / `list` / `navigate` / `aggregate` are the read operations; `create` /
`update` / `delete` / `invoke` are the write/derived defaults; **`manage`** is
an *elevated* verb reserved for operations that require owner-equivalent
authority (schema / policy changes). The point of `manage` is that a policy can
grant `invoke` without granting `manage` — so an action that declares
`operation: 'manage'` is callable only by principals whose policy resolves
`manage` (owner and admin do; a plain **member** does not — `manage` is excluded
from the member read/run baseline). It's purely additive: an action that omits
`operation` keeps the name-derived behaviour exactly.

```json
{
  "name": "billing_config.rotate_key",
  "description": "Rotate the org billing key — owner-equivalent authority.",
  "object_type": "billing_config",
  "input": { "config_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent"],
    "operation": "manage"
  },
  "idempotency": "none",
  "handler": {
    "steps": [
      { "op": "update", "object": "billing_config", "where": { "id": "$input.config_id" }, "set": { "key_rotated_at": "$now" } }
    ],
    "returns": { "config_id": "$input.config_id" }
  }
}
```

**Why this matters — closing a refine bypass.** `catalog.apply` (the bundle
apply that re-shapes the catalog) is gated at `manage`, so a member can't
re-declare the catalog. But `catalog.refine_object_type` is itself a catalog-
shaping action — and by name it would derive the coarse `invoke` default, which a
member *can* resolve. Without an override a member could therefore bypass the
`catalog.apply` manage-gate by calling `refine_object_type` directly. So
`refine_object_type` declares `governance.operation: 'manage'`: a direct refine
now demands the same owner-equivalent authority as the apply path, and a member
is denied at the gate. The general lesson: when an action's *effect* is more
privileged than its name implies, pin the operation it gates on explicitly rather
than relying on the name-derived default.

## `target` — the external-write scope target (ADR-0005)

An ActionType may declare an optional `target` (`WriteTargetSchema` in
`packages/canon/src/actions/actions/declare.ts`) that tells the **external write
gate** which row a per-customer `row_filter` is scope-checked against — used when
the action's target row is NOT identified by a plain `payload.id` (a
derived-from-actor or by-reference write). It has two fields:

- `object_type` — the registered ObjectType whose row the grant's `row_filter`
  is evaluated against.
- `by` — how the concrete target id is resolved: `"id"` (the default `payload.id`),
  `{ "input_key": "<field>" }` (the id comes from an input field), or
  `{ "actor_attr": "<attr>" }` (the id is the server-derived `:actor.<attr>`, an
  ADR-0004 bound attribute the client can never supply). The handler must then
  address the row by the `$actor.<attr>` write-side ref (see below), not
  `$input.*`.

```json
{
  "name": "guest.set_standing_line",
  "description": "Guest changes their own standing line.",
  "object_type": "guest",
  "input": {
    "guest_id": { "type": "string" },
    "line": { "type": "string" }
  },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "handler": {
    "steps": [
      { "op": "update", "object": "guest", "where": { "id": "$actor.guest_id" }, "set": { "standing_line": "$input.line" } }
    ]
  },
  "target": { "object_type": "guest", "by": { "actor_attr": "guest_id" } }
}
```

**The handler MUST address the target row by `$actor.<attr>`, not client
input.** For an `{ "actor_attr": "<attr>" }` target, every `update`/`delete`
step on the target `object_type` must key its row by
`"where": { "id": "$actor.<attr>" }` — the server-derived attribute the gate
itself resolved. Declare **hard-rejects** (`VALIDATION`, code
`insecure_actor_attr_target`) any such step keyed by `$input.*`, a literal, a
`$<step>` ref, or with no `id` key at all — the write gate scope-checks the
*actor's own* row, so a handler that writes some *other* row would mutate a
foreign row the gate never vetted. The rejection is deliberately strict (it also
rejects the safe-but-indirect "read own row, then update by `$step.id`" shape);
the direct `$actor.<attr>` form is always available.

**Invariant — a target-declared action granted to an external app MUST be
row-filtered.** The external write gate is SKIPPED ENTIRELY for a grant with no
`row_filter`, so **declaring a `target` provides ZERO enforcement by itself** — a
target-declared action granted WITHOUT a `row_filter` reads as "scoped" but runs
UNSCOPED (and an ADR-0004 role branch that omits the filter silently unscopes
that role's writes). To stop this footgun, hosted-app **publish HARD-REJECTS**
(`validateAccessManifest`, `packages/api/src/hosted-app-publish.ts`) any
`access_manifest` grant whose `resource.action` names a target-declared action
but carries no `row_filter` — a typed `VALIDATION` error naming the action + the
`when_role` branch (each role branch is checked independently). Fix by adding a
`row_filter` that scopes the write (e.g. `id = :actor.guest_id`). If you truly
want an unconditional (all-rows) external write, DON'T declare a `target` — grants
remain the authority, and an unconditional grant on a target-less action still
admits the write.

## ActionType fields you can't declare (platform-authored only)

The `ActionType` interface (`types.ts`) carries several behavioural flags that are
**absent from `actionsDeclareInput`** — because the declare input has
`onExcessProperty: 'error'`, including any of them in your JSON fails the decode.
They exist for platform-authored actions and the auto-CRUD generators, not for
agent declarations:

| field | what it does | who sets it |
|---|---|---|
| `read` | marks a read-only action; the runtime skips its per-invoke `_outbox` change-stream row (a read is not a change event) | auto-CRUD `read`/`list` + registry read actions |
| `reaction_safe` | marks a vetted delivery action a `reactions.declare` `via` may target; `reactions.declare` rejects a `via` that isn't `reaction_safe` (closes a member-escalation vector via the cron `system` principal) | platform delivery actions only (e.g. `alerts.send`) |
| `sensitive_fields` | names input params (a token, API key, password) scrubbed to `'[redacted]'` in the audit copy; the handler still receives the real value | platform actions handling secrets (e.g. `connections.create`) |
| `persistArgs` | a pure projection that slims what the durable audit/outbox log stores (e.g. `catalog.apply` stores a bundle digest, not the ~35KB bundle) | platform actions with large inputs |

If you need read-classification or secret redaction, it's a signal the action
belongs at the platform tier, not in a `actions.declare` body.

## Idempotency postures

Declared per-ActionType (`idempotency`):

- `none` — every call runs. Good for FSM transition actions where the state
  guard is the natural retry barrier.
- `per-key` — the caller's `idempotency_key` is the cache key; a retry with the
  same key replays the first envelope verbatim. Good for destructive ops the
  caller retries on network failure.
- `natural-key` — the runtime hashes the input shape itself. Good when the
  input *is* the de-facto unique key (e.g. a "send template to X" effect).

## The transition marker

If the action drives a Phase 18.5 StateMachine transition, attach a
`transition` marker (`TransitionMarkerSchema`, also `onExcessProperty:
'error'`):

| field | required | meaning |
|---|---|---|
| `state_machine` | yes | the SM name |
| `from` | yes | `"*"` (any state) or a non-empty array of state names |
| `to` | one of to/to_expr | the constant target state |
| `to_expr` | one of to/to_expr | a DSL expression evaluated post-effects |

Exactly one of `to` / `to_expr` must be present — the schema rejects a marker
with neither (`"transition must declare either \`to\` or \`to_expr\`"`).

Carrying a marker buys the action three slabs of machinery for free:
a pre-handler guard (`INVALID_STATE` if the row isn't in `from`), an
auto-injected post-handler state write to `to`, and `on_enter`/`on_exit` hook
enqueues. **The handler must NOT write the FSM-bound property itself** —
`actions.declare` rejects it with `VALIDATION/direct_state_write`.

## A complete declaration (input + envelope)

```json
{
  "name": "task.start",
  "description": "Move an open task into in_progress and stamp the start time.",
  "object_type": "task",
  "input": { "task_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Task.status",
    "from": ["open"],
    "to": "in_progress"
  },
  "handler": {
    "steps": [
      { "op": "read", "object": "task", "where": { "id": "$input.task_id" }, "as": "task" },
      { "op": "update", "object": "task", "where": { "id": "$input.task_id" }, "set": { "started_at": "$now" } }
    ],
    "returns": { "task_id": "$input.task_id" }
  }
}
```

Expected envelope on success:

```json
{
  "data": { "name": "task.start", "registered": true },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

Note: the handler reads/updates `task` but never writes `status` — the runtime
auto-writes `status='in_progress'` from the transition marker. `actions.declare`
is itself `per-key`: re-submitting byte-identical JSON is a cached replay or
no-op; submitting *different* JSON under the same name fails with
`INVALID_STATE/declaration_changed` (use `actions.update` for deliberate
changes).

## See also

- [actions-handler-patterns.md](actions-handler-patterns.md) — the handler DSL body.
- Base skill: `canon skill --name canonify resource declaring-actions.md`.
