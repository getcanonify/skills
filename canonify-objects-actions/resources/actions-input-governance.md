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
| `presentation` | no | optional HUMAN presentation block (label/group/emphasis/confirm/…) for a generated UI — see below |

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

## `target` — the external-write scope target (docs/adr/0005)

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
  docs/adr/0004 bound attribute the client can never supply). The handler must then
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
UNSCOPED (and an docs/adr/0004 role branch that omits the filter silently unscopes
that role's writes). To stop this footgun, hosted-app **publish HARD-REJECTS**
(`validateAccessManifest`, `packages/api/src/hosted-app-publish.ts`) any
`access_manifest` grant whose `resource.action` names a target-declared action
but carries no `row_filter` — a typed `VALIDATION` error naming the action + the
`when_role` branch (each role branch is checked independently). Fix by adding a
`row_filter` that scopes the write (e.g. `id = :actor.guest_id`). If you truly
want an unconditional (all-rows) external write, DON'T declare a `target` — grants
remain the authority, and an unconditional grant on a target-less action still
admits the write.

**The `row_filter` must also BIND the caller, not just be present.** The
`row_filter` grammar accepts constant-only clauses (e.g. `deleted = 0`,
`status = 'active'`), and a filter that compiles is not automatically a filter
that *scopes*. For a target-declared action, publish additionally rejects a
grant whose `row_filter` compiles but never binds the caller — a filter made
of literals only (`deleted = 0`) satisfies the presence check above while
scoping nothing, letting every external actor write every matching row. Scope
the write to the caller (`id = :actor.guest_id`, `id = :actor_id`, or
`:principal.*` where applicable) — literals may still appear, ANDed alongside
a binding clause (e.g. `guest_id = :actor.guest_id AND deleted = 0`).

## `presentation` — the human-facing block (ADR-free, tick iwd)

An optional block (`ActionPresentation`, mirrored by `PresentationSchema` in
`declare.ts`, `onExcessProperty: 'error'`) that tells a generated UI how to
label, group, style, and confirm this action. Every field is optional and
purely additive. `presentation.label` is the canon author's action label; an
action that omits it keeps the compatibility fallback described below.

```json
{
  "presentation": {
    "label": "Rotate key",
    "group": "Security",
    "icon": "key-round",
    "emphasis": "secondary",
    "destructive": true,
    "confirm": "Rotate the billing key? Existing integrations using the old key will break.",
    "success": "Key rotated.",
    "refresh": "refresh"
  }
}
```

| Field | Type | Meaning |
|---|---|---|
| `label` | string | Button/menu caption. |
| `description` | string | One-line human blurb, distinct from the agent-facing `description`. |
| `group` | string | Grouping/section key for menus and toolbars. |
| `icon` | string | Lucide icon name. |
| `emphasis` | `'primary' \| 'secondary'` | Visual prominence — closed literal, strictly validated. |
| `destructive` | boolean | Styles the control as dangerous; pairs with `confirm`. |
| `confirm` / `success` / `pending` | string | Confirmation / success-toast / approval-pending copy. |
| `refresh` | `'refresh' \| 'navigate' \| 'none'` | Post-success hint — closed literal. |
| `inputLabels` | `Record<string, string>` | Action-owned human labels for input parameters, keyed by input name. Values may be `@t:` keys. |
| `inputValueLabels` | `Record<string, Record<string, string>>` | Action-owned human labels for enum input tokens, keyed by input name and raw token. Values may be `@t:` keys. |

Threaded onto the built ActionType by `compileActionType` (the same choke
point every path — declare, update, and boot-restore — runs through, so
`presentation` can't silently get stripped on one path and not another), and
projected by discovery (`summariseAction`) so REST/CLI/MCP all see it.

**Presentation is display-only** — it never gates authorization; `hidden`
lives on the *override* layer below, not here, and even a hidden action stays
invokable over CLI/REST/MCP.

**Precedence against `catalog.refine_object_type`'s `display.actions`.** An
ObjectType's `display.actions` map (keyed by the fully-qualified action name)
can *override* `label`/`emphasis`/`hidden` for one action **without touching
the action's own declaration** — see
[`objecttype-refinement.md`](../../canonify-app-authoring/resources/objecttype-refinement.md)
(base skill: `declaring-objecttypes.md`). When both exist, the object-type
override's `label`/`emphasis` win over this block's, and `hidden` (which only
the override carries) drops the action from the surface entirely; grouping
always comes from `presentation.group` (the override has no `group` field).
Use `presentation` for the action's own baseline appearance and
`display.actions` for a per-ObjectType tweak (e.g. relabeling one verb, or
demoting it into the overflow menu) without redeclaring the action.

### Localization ownership and fallbacks

**Action labels** are canon-owned: use `presentation.label`, an ObjectType
`display.actions` override, or a ViewSpec `label` / `submitLabel`. Each can be a
literal or an `@t:<key>` locale reference. For generated CRUD, the platform may
localize the verb and combine it with the author's localized ObjectType display
name. If neither is supplied, the acronym-aware English humanization of the
action id is the legacy fallback.

This is distinct from **fixed platform chrome** such as `Search`, `Filter`,
empty-state frames, and form scaffolding, which comes from the platform locale
catalog and falls back to English. **Generated-from-catalog** field labels
split by writer, because a custom action's parameters are its own vocabulary
rather than the ObjectType's columns. For an **auto-CRUD writer** — only
`<type>.create`, `<type>.update` and `<type>.replace`, whose inputs ARE the
ObjectType's columns — the label comes from the property's `label` and falls
back to English humanization.
**Custom-action** parameter labels come from this block's `inputLabels`; their
enum input values come from `inputValueLabels`. Auto-CRUD enum values continue
to come from the property's `valueLabels`. Each declaration falls back to
English humanization only when its own label is absent or unresolved. The
platform must not infer tenant vocabulary by translating identifiers. See the
the shared string ownership contract (DESIGN-SURFACE.md §2.4).

**`actions.update` preserves `presentation` on omit.** An update submits the
*full* declaration, but since `presentation` is a display refinement rather
than the action's operative shape, an update that leaves it out **keeps** the
previously-declared block instead of wiping it — the same preserve-on-omit
posture the provenance partition already gets. Submit `"presentation": {}`
explicitly to clear it.

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

Declared per-ActionType (`idempotency`), and now **enforced** by the runtime
(tick 9mu) — the declared posture is not advisory:

- `none` — every call runs; no caller-key dedup exists for this action.
  Good for FSM transition actions where the state guard is the natural retry
  barrier. If a caller supplies `idempotency_key` anyway, the runtime
  **ignores it** — the key is inert, not an error, and the action runs
  exactly as its declared posture already promises. Ignore, not reject,
  because internal callers unconditionally thread a synthesized
  `idempotency_key` into every invoke they drive regardless of the TARGET
  action's declared posture — the Schedule executor above all
  (`packages/canon/src/schedules/executor.ts`) keys every scheduled fire
  `sched:<name>:<row>:<fire_window>` as its own fire-window dedup, and a
  schedule may target *any* action, `none`-postured FSM transitions included
  (exactly the case the docs recommend scheduling). Rejecting here would fail
  every such schedule; a caller relying on the key for an action that doesn't
  support it instead gets an envelope where the key simply did nothing —
  still a real gap, but one that surfaces as a correctness bug in testing
  (two rows where one was expected) rather than a routine internal pattern
  breaking outright.
- `per-key` — the caller's `idempotency_key`, scoped to `(action_id,
  principal_id)`, is the ownership key for an atomic receipt: exactly one
  caller executes; a same-principal retry with the same key **replays the
  first envelope verbatim**; the same key with **different** typed input
  returns `CONFLICT` / `IDEMPOTENCY_KEY_REUSED` instead of executing or
  replaying; a still-running owner answers a concurrent retry with `CONFLICT`
  / `IDEMPOTENCY_IN_PROGRESS`. Replayed envelopes are pruned after **7 days**,
  so a retry with the same key later than that re-executes rather than
  replays. Good for destructive ops the caller retries on network failure.
  This is the **only** posture the caller-key receipt backs — every other
  posture ignores a supplied key (see `none` above).
- `natural-key` — **the runtime implements no caller-key mechanism for this
  posture.** There is no input-shape hash or projection today; a caller
  supplying `idempotency_key` on a `natural-key` action gets the same
  ignore-the-key behavior as `none` — the key does nothing, and dedup lives
  entirely inside the handler, as an ad hoc lookup-before-insert
  against whatever the author considers the natural key (e.g.
  `knowledge.observe` / `knowledge.contribute` look up an existing row by
  `(org_id, scope_kind, scope_value, scope_row_id, title)` before inserting).
  This has a real gap: without a DB uniqueness constraint backing that lookup,
  two concurrent callers can both miss the lookup and both insert — the same
  race class the `per-key` atomic receipt (tick dgs) exists to close, just not
  closed here. A durable, deterministic `natural-key` projection (backed by a
  DB constraint, not a handler convention) is a real product/schema decision
  Canonify has not made; do not treat this posture as "hashed and safe" until
  it is.

### `__bypass_runtime_tx` actions carry a weaker guarantee

An action declaring `__bypass_runtime_tx: true` (e.g. `catalog.apply`) opens
its **own short transactions per reconcile sub-step** instead of running
inside the runtime's one interactive transaction — deliberately, so a
long-running apply survives Turso's idle-transaction cutoff (see
`packages/canon/src/actions/catalog/apply.ts`). That breaks the receipt's
usual co-commit story:

- On the normal (non-bypass) `per-key` path, the receipt is settled **inside**
  the same transaction as the mutation, so "the receipt says committed" and
  "the mutation is committed" are the same fact.
- On a bypass action, the mutations land across **multiple already-committed
  sub-step transactions** before the receipt is ever settled. If the receipt's
  lease is reclaimed by another attempt after that work has already landed,
  the runtime cannot roll the sub-steps back — it logs the fenced reclaim and
  returns the envelope the work actually produced.

So `per-key` on a `__bypass_runtime_tx` action still prevents a second caller
from *starting* redundant work while the first is in flight, and still
replays a settled envelope on retry — but it cannot promise that two
concurrent same-key attempts produce **exactly one** set of committed
sub-step changes the way a normal transactional action can. Treat a
bypass action's idempotency as "best-effort, sub-step granular," never as the
same atomic guarantee the receipt gives everything else.

### The single-statement fast path carries the same weaker guarantee

`__bypass_runtime_tx` is not the only carve-out. An action whose whole effect
is one statement (`singleStatementWrite` — the auto-CRUD `create`/`update`/
`delete` generator uses it for every plain, non-transitioning object, i.e.
the flagship `per-key` actions most callers actually invoke) sends that
statement and its outbox row as **one `batch()`**, with no interactive
transaction opened at all (`packages/canon/src/runtime.ts`, "SINGLE-STATEMENT
FAST PATH"). Because there is no transaction, the receipt cannot co-commit
with the mutation here either:

- The batch (mutation + outbox row) commits first.
- The receipt is settled **after**, in a separate statement, exactly as on
  the `__bypass_runtime_tx` path.
- A crash between the batch commit and that post-hoc settle leaves the
  receipt `in_progress`. If its lease expires before a retry, a same-key
  retry reclaims the receipt and executes a **second** committed mutation —
  the mutation itself is not idempotent from the receipt's point of view; it
  ran twice because the receipt never got to say otherwise.

So a plain auto-CRUD create/update/delete carries this window whether or not
its action ever declares `__bypass_runtime_tx`. Treat "receipt settled inside
the same transaction as the mutation" as true only on the path that opens an
interactive transaction at all (a transition plan, or a handler with neither
`singleStatementWrite` nor `__bypass_runtime_tx`) — both other paths settle
the receipt post-commit and share this same reclaim-after-crash window.

### External effects are at-least-once, never exactly-once

The carve-outs above are all about a **local** receipt: whether "the receipt
says committed" and "the mutation is committed" are the same fact inside
Canon's own database. An external effect — an email, a webhook — breaks that
question entirely, on every path, `per-key` included: a local receipt cannot
roll back a message that has already left the building. If the process
crashes after the send succeeds but before the receipt settles, a retry (the
outbox drain's re-invoke, or a same-key caller retry) re-executes the handler
and sends **again**. There is no version of the receipt that closes this
window, because the thing it would need to roll back lives in someone else's
system.

What Canon can do instead is give the receiving side a **stable token** to
dedup on, so a resend of the same effect is at least *recognisable* as a
resend. For an action the outbox drain re-invokes from an `_outbox` row
(`kind='effect'` — an FSM `on_enter`/`on_exit` hook, `on_fail` compensation,
or a Reaction/schedule fan-out), the drain merges the row's own identity
(`idempotency_key ?? id`) into the invoked action's args as
`idempotency_key` (`packages/canon/src/outbox-processor.ts`,
`mergeIdempotencyKey`) — a stable value that is the SAME across every retry
of that row. A plain `Schema.Struct` action input strips unknown keys on
decode, so this is a silent no-op for an action that doesn't declare the
field; declaring it is how an action OPTS IN to receiving it. Whether that
token becomes an actual dedup guarantee downstream depends entirely on the
provider on the other end — Canon is honest about that per kind, not
uniform:

- **`email.send_template`.** The token is forwarded to `EmailSink.send` as
  `EmailMessage.idempotency_key`. The one production binding
  (`packages/api/src/email-sink.ts` over Cloudflare Email Routing's
  `send_email` binding) has **no idempotency/dedup API of its own** — it is a
  raw MIME relay. The binding's only move is to reuse the token as the MIME
  `Message-ID` header instead of minting a fresh one per attempt, so a
  redelivery of the same row carries the same `Message-ID`. That is a
  best-effort courtesy to whatever MTA/mailbox receives it — nothing in this
  path promises to dedup on a repeated `Message-ID`. Email delivery through
  this action is at-least-once, full stop; a duplicate send is possible and
  the receiving side is not guaranteed to collapse it.
- **`forms.webhook.deliver`.** The token is sent as the
  `X-Canon-Idempotency-Key` header (alongside the existing `X-Canon-Signature`
  HMAC) on every POST. Here the "provider" is an arbitrary customer-owned HTTP
  endpoint, not a named vendor API — Canon can offer the header, but cannot
  make the receiver honour it. A webhook receiver that wants exactly-once
  effects on its own side should dedup incoming requests by this header;
  Canon's guarantee stops at "the same row always sends the same value."
  Delivery through this action is at-least-once regardless of whether the
  receiver uses the header.

**Do not read either of these as exactly-once.** A `per-key` idempotency
posture on `email.send_template` or `forms.webhook.deliver` still protects
Canon's OWN receipt/audit state exactly as described above — it stops a
second caller from *starting* redundant work while the first is in flight,
and replays a settled envelope on retry. It does not, and cannot, retract an
email already sent or a webhook already POSTed. If you are declaring a new
action that drives an external side effect, the honest default is
at-least-once; only claim more once the concrete downstream binding actually
backs it, and say so explicitly, the way the two cases above do.

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
