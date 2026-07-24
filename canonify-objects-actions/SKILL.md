---
name: canonify-objects-actions
description: Deep-dive reference + tutorial for declaring ObjectTypes and ActionTypes in Canonify — typed object shape, property visibility, links/cardinality/on_delete, governance, idempotency, transition markers, and the handler DSL.
version: 1.0.0
---

# Canonify — Objects & Actions

This skill goes one level deeper than the base **canonify** manual on the two
verbs you spend the most time with: `catalog.refine_object_type` (the shape of
your data) and `actions.declare` (the governed writes over it). Read the base
skill first for the platform tour and the runtime frame:

```sh
canon skill --name canonify instructions
```

Everything here is grounded in the real surfaces — the effect/Schema
validators in `packages/canon/src/handler-dsl/schema.ts`, the property/link
types in `packages/canon/src/types.ts`, and the action input schemas in
`packages/canon/src/actions/`. Every JSON recipe below decodes against those
live schemas (a unit test asserts it — see the bottom of this page).

The examples are deliberately **domain-agnostic** — generic `customer`,
`order`, `task`, `counter` shapes. None of them commit to any specific product
model; they're patterns you adapt.

---

## Part 1 — ObjectTypes: the typed shape

An ObjectType is the typed view over one table. Auto-CRUD generates a
permissive default (one `mapped` `string` per column, no links, no
description); you `catalog.refine_object_type` it into the rich domain shape.

### Property kinds

Three kinds exist (`types.ts`): `mapped` (a real column, writable), `derived`
(a SQL expression in the read view, read-only), and `computed` (a synchronous
function over the row, never persisted). **Only `mapped` is refinable from an
agent surface** — `ObjectPropertyRefinement` requires `kind: "mapped"`.
Refining a `derived`/`computed` property fails `VALIDATION`.

### The seven PropertyTypes

`string` · `number` · `boolean` · `timestamp` · `json` · `enum` · `file`.
`timestamp` is an ISO-8601 UTC string (never an epoch int); `enum` is a string
narrowed to a `values` set; `file` holds a `_files` id as TEXT.

### Enum narrowing — the load-bearing refinement

A state column must be a typed `enum` before a StateMachine can bind to it.
Narrowing is validated against persisted data:

```json
{
  "name": "task",
  "description": "A unit of work assigned to a member.",
  "properties": {
    "status": {
      "kind": "mapped",
      "column": "status",
      "type": "enum",
      "values": ["open", "in_progress", "done", "cancelled"]
    }
  }
}
```

Expected envelope (success):

```json
{
  "data": { "name": "task", "refined": true, "refinement_id": "re_…" },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

The validator (`validateRefinement`) enforces: the base must be `string` or
`enum`; `values` must be non-empty; and **every value already in the column
must be covered** — else `"…cannot be narrowed to enum […] — existing row has
value "…""`. Tightening `nullable: false` requires zero NULL rows. Full rules:
[resources/object-property-types.md](resources/object-property-types.md).

### Links, cardinality, on_delete

A link is a typed edge to another ObjectType:

```json
{
  "name": "customer",
  "links": {
    "tasks": { "target": "task", "cardinality": "many", "via": "customer_id", "on_delete": "cascade" },
    "orders": { "target": "order", "cardinality": "many", "via": "customer_id", "on_delete": "restrict" }
  }
}
```

```json
{
  "data": { "name": "customer", "refined": true, "refinement_id": "re_…" },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

`via` is the FK column **on the target table** (the most common mistake is
naming the source column). `cardinality` is `one`|`many`. `on_delete` is
**data semantics, not handler semantics** — `restrict` (default; refuse with
`INVALID_STATE` if children exist), `cascade` (delete children first), or
`set_null` (null the FK; requires a nullable `via` column). It is honoured by
*every* delete path: auto-CRUD's `<obj>.delete`, a declared delete step, and a
manual `objects.delete`. The validator rejects an unregistered `target` or a
`via` column missing on the target table. Full contract:
[resources/object-links-and-cardinality.md](resources/object-links-and-cardinality.md).

### Compound objects — `part_of`, `rollups`, `parts_frozen_when`

Marking a link `part_of: true` makes the target a constituent **part** of the
root's identity (an `invoice` ⊃ its `lines`), read/written/governed as one unit
(ADR 0002). On the link it carries a forest constraint and a small set of shape
rules — full list in
[resources/object-links-and-cardinality.md](resources/object-links-and-cardinality.md).
Two more `catalog.refine_object_type` fields decorate a compound **root**:

- **`rollups`** — declared root summary properties. A map of result-property name
  → `{ link, agg, column? }` over one of the root's `part_of` child links. The
  runtime computes each at read time (a policy-gated aggregate over admitted
  child rows) and merges it onto the row in `get`, `list`, and `--expand`, so you
  declare `invoice.total` once instead of restating the navigate-aggregate at
  every call site. `agg` is `sum`|`count`|`avg`|`min`|`max`; the numeric ops
  require a numeric `column` on the child, `count` takes none. Validated at
  `catalog.apply` against the `part_of` graph — diagnostic `rollup_invalid` when
  `link` is unknown / not a `part_of` link on this root, or the column is
  missing/unknown/non-numeric.

```json
{
  "name": "invoice",
  "links": {
    "lines": { "target": "invoice_line", "cardinality": "many", "via": "invoice_id", "on_delete": "cascade", "part_of": true }
  },
  "rollups": {
    "total": { "link": "lines", "agg": "sum", "column": "amount" },
    "line_count": { "link": "lines", "agg": "count" }
  }
}
```

- **`parts_frozen_when`** — a lifecycle freeze gate. Shape
  `{ column, in: [...] }`: when the root row's `column` value is in the `in` set,
  the root's `part_of` parts are **immutable** — any insert/update/delete of a
  part inside a compound write fails with a `CONFLICT` envelope
  (`details.reason === "FROZEN"`). `column` must be a `mapped` property on the
  root; `in` must be non-empty. Validated at `catalog.apply` — diagnostic
  `parts_frozen_invalid` when the type has no `part_of` link, the column isn't a
  mapped root property, or `in` is empty. A *born-frozen* create succeeds (the
  gate blocks later part mutations, not the initial cluster creation).

```json
{
  "name": "invoice",
  "links": {
    "lines": { "target": "invoice_line", "cardinality": "many", "via": "invoice_id", "on_delete": "cascade", "part_of": true }
  },
  "parts_frozen_when": { "column": "status", "in": ["locked", "paid"] }
}
```

### Visibility — `hidden` vs `confidential`

Two distinct mechanisms:

- **`hidden`** (property-level, refinable) — withhold a column from the typed
  read shape, or surface a platform-hidden column (e.g. `org_id`) so a handler
  can write through it. A property-shape toggle, **not** an auth boundary — a
  handler that names a hidden property can still use it.
- **`confidential`** (ObjectType-level, **platform-tier**, absent from the
  refinement input) — marks a type as holding sensitive data: it default-denies
  without an explicit grant, and a *granted* read/invoke writes its audit row
  with `status='sensitive'` so access is reviewable via `canon audit --status
  sensitive`.

Detail and the auth-context story: [resources/object-visibility.md](resources/object-visibility.md).

---

## Part 2 — ActionTypes: governed writes

`actions.declare` registers a custom ActionType. The handler DSL is the body;
the declaration wraps it with name, input descriptor, governance, idempotency,
and an optional transition marker. Unknown top-level fields fail the decode
(`onExcessProperty: 'error'`).

### Input, governance, idempotency

- **input** — a flat record of field → descriptor. `type` is `string` |
  `number` | `boolean` | `json` (no `timestamp`/`enum`/`file` at the input
  layer; an enum input is `enum: [...]` over a `string`). Optional flags:
  `prefix`, `min`, `max`, `enum`, `format` (`email`|`iso8601`), `optional`,
  `nullable`.
- **governance** — `requires_approval` (bool) + `allowed_principals`
  (non-empty array of `user`|`agent`|`service_account`), with optional `policy`
  and `requires_approval_when` (a where-DSL predicate over the parsed input
  that conditionally requires approval, e.g. `"amount_cents > 100000"`).
- **idempotency** — `none` (every call runs; right for FSM transitions where
  the state guard is the retry barrier), `per-key` (the caller's
  `idempotency_key` replays the first envelope), or `natural-key` (the runtime
  hashes the input shape).

### The subject — which input is the operated-on object

An action that operates on a specific object carries that object's id as an
input — its **subject**. The catalog derives it from the `<type>_id` convention:
if the action declares an input named `` `${object_type}_id` `` (`task.start` →
`task_id`), `subject = { param: "task_id", object_type: "task" }`. You may set
`subject` **explicitly** on the ActionType to override the convention or
disambiguate a multi-FK action (name a non-conventional param, or pick the FK
that is the real subject); explicit always wins. An action with no `<type>_id`
input — `<type>.create`, or anything that doesn't operate on one existing row —
has no subject.

`subject` is surfaced in discovery (`DescribedAction.subject`) over REST / CLI /
MCP, so every surface — and the web UI — knows which input is the operated-on
object instead of re-guessing the convention. The web renderer uses it to
**auto-bind** the subject from render context (a record detail, a list row, a
ViewSpec button in a record-scoped view), so app authors don't hand-wire
`presetParams: { <type>_id: "$context.id" }`. `subject` is metadata + a UI
binding rule only — it is **not** a server-side auto-inject: the runtime
`invoke` still receives the subject as an ordinary caller-supplied input, so
stateless CLI / MCP / REST callers pass it explicitly.

### A verb binds its own constant — don't expose a raw state column as input

A user-facing action is a **verb with intent** (`flag_at_risk`, `archive`,
`approve`), not a generic setter for a column. So the action should bind the
state it implies as a **constant in the handler**, and take only the subject as
input — never surface the underlying state column as a required free-form field.

The trap: modelling "set X" as a single action whose input *is* the column
(`set_at_risk` taking `at_risk: number`, "1 to flag, 0 to clear"). One verb
covers both directions, but the auto-form then renders a required free-form box
the user must fill — it can't be one-tap, and on the web surface a
required-but-empty field is rejected by the decoder as a `VALIDATION/400` the
user reads as an opaque failure. Splitting into two zero-input verbs
(`flag_at_risk` → handler `set: { at_risk: 1 }`, `clear_at_risk` →
`set: { at_risk: 0 }`) makes each a one-tap button whose subject the toolbar
presets, with nothing to type.

Heuristic: if an action's input is a raw state/enum column with a "pass N to
mean X" docstring, split it into one verb per intent (or, for a true free value
like a price, keep the input but give it a sensible default). The subject is the
one input a UI-invoked action should need. (Ticket jbu.)

### Transition markers

If the action drives a StateMachine transition, attach a `transition` marker
(`state_machine`, `from` = `"*"` or a non-empty array, and exactly one of `to`
/ `to_expr`). The marker buys a pre-handler guard (`INVALID_STATE` off the
`from` set), an auto-injected post-handler state write to `to`, and
`on_enter`/`on_exit` hooks — all for free. (`"*"` also onboards a row whose
FSM property is still NULL; an explicit `from` array never matches NULL.)
**The handler must not write the FSM-bound property itself**
(`VALIDATION/direct_state_write`).

**Non-conventional id input?** The guard + auto state-write locate the row's
id the same way the subject binding does: an explicitly declared `subject`
naming this ObjectType binds its `param` as the id input, overriding the
`<type>_id`/`id` naming convention. So a transition action whose id param is
non-conventional just needs a declared `subject` — you don't have to rename
the public input. Without a conventional id input *and* without a declared
subject, registration is rejected (`no_id_input`).

### A complete declaration

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
  "transition": { "state_machine": "Task.status", "from": ["open"], "to": "in_progress" },
  "handler": {
    "steps": [
      { "op": "read", "object": "task", "where": { "id": "$input.task_id" }, "as": "task" },
      { "op": "update", "object": "task", "where": { "id": "$input.task_id" }, "set": { "started_at": "$now" } }
    ],
    "returns": { "task_id": "$input.task_id" }
  }
}
```

Expected envelope (success):

```json
{
  "data": { "name": "task.start", "registered": true },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

The handler reads/updates `task` but never writes `status` — the runtime
auto-writes `status='in_progress'` from the marker. Full envelope reference:
[resources/actions-input-governance.md](resources/actions-input-governance.md).

---

## Part 3 — The handler DSL

The handler is a linear sequence of typed CRUD steps glued by the `$ref`
grammar: `$input.<field>`, `$<step>.<field>`, `$now`, `$principal.id|org_id|
kind`, `$actor.<attr>` (a server-set external-principal attribute, for
`actor_attr` write targets). A value-position string starting with `$` MUST be a valid ref
(lowercase-snake head + dot segments); `$Input.x` (capital) is rejected at
decode; escape a literal `$` as `$$`.

Step kinds: `read` · `read_optional` · `read_aggregate` · `insert` · `update` ·
`delete` · `assert`. Each may carry a `when` guard predicate (mandatory on
`assert`). Optional top-level `returns` block and `on_fail` compensator.

### Multi-table atomic write

```json
{
  "steps": [
    { "op": "read", "object": "lead", "where": { "id": "$input.lead_id" }, "as": "lead" },
    { "op": "insert", "object": "customer", "values": { "name": "$lead.name", "email": "$lead.email", "org_id": "$principal.org_id" }, "as": "customer" },
    { "op": "insert", "object": "contact", "values": { "customer_id": "$customer.id", "name": "$lead.name", "email": "$lead.email", "role": "primary" }, "as": "contact" },
    { "op": "update", "object": "lead", "where": { "id": "$input.lead_id" }, "set": { "customer_id": "$customer.id", "converted_at": "$now" } }
  ],
  "returns": { "customer_id": "$customer.id", "contact_id": "$contact.id" }
}
```

All steps run in one transaction; any failure rolls everything back. `insert`
stamps `id`/`created_at`/`updated_at` automatically.

### Predicates, find-or-create, assert

A predicate is a `ref` plus **exactly one** operator: `eq` · `neq` · `gt` ·
`gte` · `lt` · `lte` · `in` · `present` · `absent`. No `and`/`or`/`not` — chain
`assert` steps for conjunction. `read_optional` binds `null` (not `NOT_FOUND`)
on zero rows, enabling find-or-create with a `present`/`absent` `when` guard.
`assert` aborts the tx with your verbatim `else.message` (code restricted to
`VALIDATION`|`FORBIDDEN`).

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "assert", "when": { "ref": "$order.status", "eq": "open" }, "else": { "code": "VALIDATION", "message": "order must be open to add items" } },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "item_count": { "op": "+", "left": "$order.item_count", "right": 1 } } }
  ]
}
```

### Aggregates and expressions

`read_aggregate` takes exactly one aggregator (`count: true` or
`sum`/`avg`/`min`/`max` with a column name), binds under `$<as>.<op>`, and sees
same-tx rows. `set`/`values` may hold a flat binary expression — arithmetic
(`+ - * /`) or date (`date_add`/`date_sub` with a duration token like `3d`,
optionally `business_days` against a named `calendar` ObjectType). Expressions
are flat in v1 (no nested operands). All recipes, plus the `on_fail`
compensator and the gotchas the typecheck pass rejects:
[resources/actions-handler-patterns.md](resources/actions-handler-patterns.md).

---

## Part 4 — Platform action surfaces you invoke (not declare): e-signature

Not every action you call is one you authored. The platform ships a set of
**built-in** ActionTypes — registered unconditionally in the catalog, not
through `actions.declare` — that you **invoke** like any other action. The
largest such surface is **e-signature**: a complete native signing primitive
(eIDAS SES — identity-bound + tamper-evident) over three object types
(`signature_request`, `signature_envelope`, `signature`) plus a per-member
reusable-signature store (`member_signature`).

You author none of these; you call them. The 16 actions cover the single-signer
lifecycle (`signature_request.create` / `.send` / `.view` / `.sign` /
`.decline`), trust checks on the signed record (`signature.verify` /
`.certificate`), the ordered multi-signer envelope (`signature_envelope.create`
/ `.send` / `.countersign` / `.download` / `.my_pending_countersigns` /
`.decline` / `.void`), and the saved member signature
(`member_signature.set` / `.get`).

Three things make this surface different from the domain actions you declare, and
they recur across every signing action:

- **Two principal populations.** External signers (a non-member kind, reached by
  an emailed magic link) authenticate by redeeming a `link_token`; org members
  (`user`/`agent`) authenticate by their session. The external kind is exempt
  from the per-action `allowed_principals` gate, so each external-only handler
  carries its own kind-guard.
- **`link_token` is a bearer credential**, and it is `persistArgs`-**redacted**
  from the audit/outbox args so a leaked audit row can never replay a still-valid
  token. Signing **consent** (`consent.acknowledged` + a known `consent_version`)
  is a mandatory, server-resolved intent affirmation, and the WHO/WHERE evidence
  (IP / User-Agent / `auth_method` / session) is **server-derived** off the
  `RequestContext` seam — never a client input you can pass.
- **Terminal lifecycles propagate.** A request freezes its parts once terminal
  (`parts_frozen_when`); declining any envelope party ends the whole envelope
  (siblings swept `voided`); a sender `void` clears the countersign queue;
  envelope completion is **automatic** (there is no `signature_envelope.complete`
  action) when the last slot signs.

The full action-by-action input shapes, status enums, and scope rules are in
[resources/esign-actions.md](resources/esign-actions.md). That page is the
**action surface an agent invokes**; the browser-portal side (the external
`/_sign` page, magic-link + PIN session, and the `/_canon/sign-token` +
`/_canon/download-token` proxy endpoints) is the operations doc
`repo-wiki/operations/signing-protocol.md`.

---

## Discoverable identically on every surface (isomorphism)

This skill is served verbatim — **byte-identical** — on all three transports.
The runtime reads the same files for each; there is no per-surface rendering of
skill content. Pick whichever matches the surface you're driving:

| | REST | CLI | MCP |
|---|---|---|---|
| manifest | `GET /v1/skills/canonify-objects-actions` | `canon skill --name canonify-objects-actions` | the skill resource list |
| instructions | `GET /v1/skills/canonify-objects-actions/instructions` | `canon skill --name canonify-objects-actions instructions` | served from the same loader |
| a resource | `GET /v1/skills/canonify-objects-actions/resources/<path>` | `canon skill --name canonify-objects-actions resource <path>` | the named resource |

All three resolve through the same skill loader
(`packages/canon/src/skill-loader.ts`) over the same files on disk, so the
markdown an agent reads is identical no matter the transport — the same
isomorphism gate that governs action envelopes. If a fetch differed by surface,
that would be a regression caught in CI.

---

## Accuracy guarantee

Every JSON recipe in this skill — in this file and in `resources/` — is parsed
by a unit test against the live validators: handler blocks against `HandlerDsl`
(`handler-dsl/schema.ts`), refinement inputs against
`catalogRefineObjectTypeInput`, and declarations against `actionsDeclareInput`.
If a recipe drifts from the runtime grammar, CI fails. The examples here cannot
go stale relative to the code.

## Resources

- [resources/object-property-types.md](resources/object-property-types.md) — kinds, PropertyTypes, enum narrowing, nullable tightening.
- [resources/object-links-and-cardinality.md](resources/object-links-and-cardinality.md) — `target`/`via`/`cardinality`/`on_delete`.
- [resources/object-visibility.md](resources/object-visibility.md) — `hidden` + `confidential`.
- [resources/actions-input-governance.md](resources/actions-input-governance.md) — input descriptor, governance, idempotency, transition marker.
- [resources/actions-handler-patterns.md](resources/actions-handler-patterns.md) — copy-paste handler recipes + typecheck gotchas.
- [resources/esign-actions.md](resources/esign-actions.md) — the built-in e-signature action surface you invoke: the 16 `signature_request` / `signature` / `signature_envelope` / `member_signature` actions, their inputs, status enums, consent + `link_token` redaction + server-derived evidence, and the terminal lifecycle.

## Source-of-truth references

- Base skill: `canon skill --name canonify instructions` — the platform tour + runtime frame.
- [`packages/canon/src/handler-dsl/schema.ts`](../../../packages/canon/src/handler-dsl/schema.ts) — the `HandlerDsl` effect/Schema.
- [`packages/canon/src/types.ts`](../../../packages/canon/src/types.ts) — property/link/governance types.
- [`packages/canon/src/actions/actions/declare.ts`](../../../packages/canon/src/actions/actions/declare.ts) — `actions.declare` input.
- [`packages/canon/src/actions/catalog/refine-object-type.ts`](../../../packages/canon/src/actions/catalog/refine-object-type.ts) — `catalog.refine_object_type` input + validator.
