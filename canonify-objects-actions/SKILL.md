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

**Auto-CRUD and state machines.** The generated `<type>.update` and
`<root>.replace` write any mapped column — *except* a property a registered
StateMachine binds. A call that names such a property is refused on every
transport with `VALIDATION` / `details.code: direct_state_write` (the message
names the property and the machine) and writes nothing; a call that omits it
updates the other columns as usual. The check runs at invoke time, so
registering a machine on a live org protects the column immediately. To move
the state — including operator "unlock" / "cancel" — declare a transition
(next section, *Transition markers*); there is no direct-write path. Details in
[declaring-state-machines › *The bound property belongs to the machine*](../canonify/resources/declaring-state-machines.md).

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

### JSON and boolean wire contract

> **Breaking change since `2026.8.28.1409`:** For a `json`-typed property,
> INPUT = a JSON value (array or object) or a JSON string that must parse
> (otherwise `VALIDATION` names the property); STORAGE = `TEXT`; OUTPUT on
> every typed surface (action envelope,
> `objects.get`/`list`/`navigate`/`aggregate`, declared returns,
> CLI/REST/MCP alike) = the parsed value;
> `sql_read` = raw `TEXT` by contract; booleans follow the same rule
> (`0`/`1` stored, `true`/`false` returned).

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

### Localization ownership for ObjectTypes and enum values

The ObjectType declaration is the source of tenant vocabulary. **Generated-from-catalog**
field labels belong to the canon author through the property's `label`, and
ObjectType singular/plural display names belong to the author through
`display.entityKind` / `display.entityKindPlural`. Literals are shown as
declared; `@t:<key>` values resolve through the org's locale bundles. If no
label is declared, the platform's acronym-aware English humanizer is the
compatibility fallback and the localized Canon has migration debt.

For **enum values**, declare `valueLabels` beside `valueColors`, with one
literal or `@t:<key>` per value that can reach a user-facing surface. A bound
StateMachine's `StateMeta.label` is the equivalent author-owned source for its
states. If a value has no label, the platform uses the English humanized raw
value and a neutral color; it does not guess a tenant translation from the
platform locale catalog. Stored values remain stable for filters and sorting.

**Fixed platform chrome** (for example `Search`, `Filter`, empty-state frames,
and `Optional fields`) is owned by the platform's locale catalog, with English
as the fallback. **Action labels** are canon-owned through ActionType
`presentation.label`, ObjectType `display.actions`, or a ViewSpec label; an
unlabeled generated action may compose a localized platform verb with the
author's localized ObjectType name, otherwise its English action-id humanizer
is the legacy fallback. See the full four-category contract in
the platform design-surface reference, §2.4 "user-visible string ownership".

**Migration note.** Existing Canons may keep their stored names, enum values,
and action ids. Add localized `entityKind` / `entityKindPlural`, property
`label`s, complete enum `valueLabels`, and surfaced action `presentation.label`s
(or `@t:` keys with English and Danish bundle entries), then export/apply the
updated Canon. The English humanizers are compatibility fallbacks only; do not
translate generated copy into the data bundle.

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
naming the source column). `cardinality` is `one`|`many`. Set `fk_on: "source"`
only for a belongs-to link whose FK lives on the source row; it is a
navigational child→parent edge. `on_delete` is valid only on target-side
ownership links (`fk_on: "target"` or omitted), where the target table carries
the child FK. It is **data semantics, not handler semantics** — `restrict`
(default; refuse with `INVALID_STATE` if children exist), `cascade` (delete
children first), or `set_null` (null the FK; requires a nullable `via` column).
It is honoured by *every* delete path: auto-CRUD's `<obj>.delete`, a declared
delete step, and a manual `objects.delete`. A source-side link MUST omit
`on_delete`; declaration and catalog validation reject it because typedDelete
cannot enforce a policy on a navigational edge. Put the policy on the reverse
target-side link instead. The validator rejects an unregistered `target` or a
`via` column missing on the relevant FK side. Full contract:
[resources/object-links-and-cardinality.md](resources/object-links-and-cardinality.md).

### Compound objects — `part_of`, `rollups`, `parts_frozen_when`

Marking a link `part_of: true` makes the target a constituent **part** of the
root's identity (an `invoice` ⊃ its `lines`), read/written/governed as one unit
(docs/adr/0002). On the link it carries a forest constraint and a small set of shape
rules — full list in
[resources/object-links-and-cardinality.md](resources/object-links-and-cardinality.md).
Two more `catalog.refine_object_type` fields decorate a compound **root**:

The generated `<root>.replace` action can be suppressed with
`defaults: { replace: false }`. The flag is also accepted on a non-compound
type as a no-op, because no `replace` action is generated until a `part_of`
child exists.

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
- **`confidential`** (ObjectType-level, refinable) — marks a type as holding
  sensitive data: production still requires an explicit grant, denials remain
  `FORBIDDEN` with audit `status='denied'`, and a *granted* mutation writes its
  audit row with `status='sensitive'` so sensitive writes are reviewable via
  `canon audit --status sensitive`. Granted reads currently use the
  reads-denials-only audit policy and do not persist a success row.

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
  `idempotency_key` replays the first envelope, for 7 days — after that a
  same-key retry RE-EXECUTES), or `natural-key` (the runtime
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
**No declared handler may write the FSM-bound property itself** —
`actions.declare` rejects it with `VALIDATION/direct_state_write` whether or
not the declaration carries a marker (with the marker the runtime writes the
state for you; without it the write would bypass the machine). Handlers of
other verbs on the same ObjectType keep writing every *other* property.

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

## Part 3 — The handler DSL (design §§2.0–2.e)

The handler is a linear sequence of typed CRUD steps glued by the `$ref`
grammar: `$input.<field>`, `$<step>.<field>`, `$now`, `$principal.id|org_id|
kind`, `$actor.<attr>` (a server-set external-principal attribute, for
`actor_attr` write targets). A value-position string starting with `$` MUST be a valid ref
(lowercase-snake head + dot segments); `$Input.x` (capital) is rejected at
decode; escape a literal `$` as `$$`.

Step kinds: `read` · `read_optional` · `read_many` · `read_aggregate` · `insert` ·
`update` · `delete` · `assert` · `for_each`. Each may carry a `when` guard
predicate (mandatory on `assert`). Optional top-level `returns` block and
`on_fail` compensator. `read_many` binds a bounded row set for a following
`for_each`; the loop body contains the existing seven step kinds only.

Guarded and optional bindings are nullable. A step with a `when` guard still
binds its `as` name to `null` when the guard is false, and `read_optional`
binds `null` when it finds no row. A strict write-position ref in `values`,
`set`, or a bare `where` equality may therefore target only a property declared
nullable; otherwise declaration validation reports
`nullable_binding_into_non_nullable` and names the alias, its `when` guard (or
`read_optional` source), and the non-nullable target property. Predicate refs
remain null-tolerant so `present`/`absent` guards can inspect these bindings. A
`when: { ref: "$guest.id", present: true }` guard narrows `$guest` to a
present row for that guarded step only; an `assert` with the same predicate
narrowing makes the proof available to following steps. An unguarded step
after a skipped read still sees the binding as nullable.

In `read_optional`, `read_many`, and `read_aggregate` where filters, a null
binding becomes `IS NULL` and yields `null`, an empty set, or zero; `read` and
write positions remain strict and require a `when` narrowing.

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

### Bounded set fan-out (design §2.a)

Use `read_many` to bind a filtered row set, then `for_each` to run the same
body once per row. `where` must contain at least one key. By default rows are
read in `id` ascending order. Both `read_many` and `read_optional` accept one
`order_by` key (`property` plus `direction: "asc"` or `"desc"`) and a positive
`limit`; the property must be a declared number, string, or timestamp. For
`read_many`, `limit` is at most **500**. An omitted `read_many.limit` keeps the
500-row cap probe; an explicit limit bounds the result to the requested size.
For `read_optional`, ordering chooses which matching row is returned; with no
ordering it retains the existing first-row behavior, and its result is always
one row or `null`.

The set exposes one scalar, `$orders.count`: the non-null number of rows bound
for the loop (0–500). It is valid in `returns`, `values`, and `set`; name it
`rows_iterated`, not `rows_written`, because the loop body may write a
different number of rows or roll back. It is a snapshot of the bound set, not
an accumulator.

For a child collection, a set binding may also provide one bounded column
projection, but only as the direct operand of a `where` operator's `in`:
`{ "in": "$orders.id" }`. The head must name a prior `read_many`, the segment
must be a property on that set's ObjectType, and its type must match the
filtered child property. The projection becomes a bound `IN` list; an empty
parent set becomes the existing never-match `1=0` predicate, never `IN ()`.
This is the only set-property escape: `$orders.id` in equality, values, set,
predicates, returns, or `for_each.in` remains rejected, as does a bare
`$orders` outside `for_each.in`.

A cap violation or any body failure rolls back the entire action, including
writes before the loop; an empty set is a no-op. The loop alias and every body
binding are scoped to one iteration.

```json
{
  "steps": [
    { "op": "read_many", "object": "standing_order", "where": { "delivery_date": "$input.delivery_date" }, "as": "orders" },
    { "op": "for_each", "in": "$orders", "as": "order", "steps": [
      { "op": "insert", "object": "order_line", "values": { "order_id": "$order.id", "product_id": "$input.product_id", "quantity": "$input.quantity" } }
    ] }
  ]
}
```

To read a child collection without nesting `for_each`, read the parent set and
use its id projection in the child's `where.in`:

```json
{
  "steps": [
    { "op": "read_many", "object": "order", "where": { "delivery_date": "$input.delivery_date" }, "as": "orders" },
    { "op": "read_many", "object": "order_line", "where": { "order_id": { "in": "$orders.id" } }, "as": "lines" }
  ],
  "returns": { "line_count": "$lines.count" }
}
```

The second step may be an `update` with the same `where` when the child rows
need a bulk change. `$orders.count` remains the set's only scalar address;
`$orders.id` in equality, values, set, predicates, returns, or `for_each.in`
is rejected at declaration, as is a bare `$orders` outside `for_each.in`.
`for_each` cannot be nested, and its alias cannot shadow an existing binding or
the reserved heads `input`, `principal`, `now`, and `actor`.

### Predicates, find-or-create, assert (design §§2.b–2.c)

A predicate is a `ref` plus **exactly one** operator: `eq` · `neq` · `gt` ·
`gte` · `lt` · `lte` · `in` · `present` · `absent` · `contains`. Chain `assert`
steps for conjunction. `read_optional` binds `null` (not `NOT_FOUND`)
on zero rows, enabling find-or-create with a `present`/`absent` `when` guard.
`assert` aborts the tx with your verbatim `else.message` (code restricted to
`VALIDATION`|`FORBIDDEN`).

In a `when` or `assert`, `any_of` is a flat disjunction of full predicates:
each member needs its own `ref`, and `any_of` members may carry different refs.
For example, “no guest named OR named guest belongs to this customer” is
`{ "any_of": [{ "ref": "$input.guest_id", "absent": true }, { "ref": "$guest.id", "present": true }] }`.

Comparison operands are full `Value`s, so `eq`/`neq`/`gt`/`gte`/`lt`/`lte` and
each `in` member may be a literal, ref, arithmetic/date expression, or
`date_part` expression. Bare ref operands retain null-tolerant predicate
resolution; expression nodes resolve their refs strictly.

`contains` is a predicate operator for JSON-array membership. The addressed property must be
declared `type: "json"`, and its operand must resolve to a scalar (`string`,
`number`, `boolean`, `enum`, or `timestamp`). A raw JSON cell read from
SQLite is parsed before membership is checked; an already-parsed input array
is accepted. NULL means no match, while malformed JSON or a non-array value
fails with `VALIDATION`. JSON and null operands are rejected by
`contains_operand_scalar`.
For example: `when: { "ref": "$guest.tags", "contains": "vip" }`.

Every `where` position accepts a scalar Value, one operator object, or one
flat `any_of` disjunction. The closed where operator set is `gt` · `gte` ·
`lt` · `lte` · `neq` · `in` · `present` · `absent` · `contains`; literal `in`
arrays must be non-empty, and a direct `in` reference may only be a property
of a prior `read_many` set binding. The operator object carries exactly one
key. `any_of` members are scalar Values or operator objects and cannot contain
another `any_of`. A set-property `in` reference resolves to a bounded list of
bound parameters; an empty list compiles to `1=0`. `contains` compiles to
an `EXISTS` query over SQLite's `json_each`, so a JSON-array property is
required. For example, a Pax-style weekday filter uses:

```json
{
  "steps": [
    {
      "op": "read_many",
      "object": "standing_order",
      "where": {
        "weekdays": {
          "contains": {
            "op": "date_part",
            "part": "iso_weekday",
            "left": "$input.delivery_date"
          }
        }
      },
      "as": "orders"
    }
  ]
}
```

Declare Pax-style `weekdays` columns as `type: "json"`, not `type: "string"`;
`contains_requires_json` rejects string properties at declaration time.

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "assert", "when": { "ref": "$order.status", "eq": "open" }, "else": { "code": "VALIDATION", "message": "order must be open to add items" } },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "item_count": { "op": "+", "left": "$order.item_count", "right": 1 } } }
  ]
}
```

### Aggregates and expressions (design §2.d)

`read_aggregate` takes exactly one aggregator (`count: true` or
`sum`/`avg`/`min`/`max` with a column name), binds under `$<as>.<op>`, and sees
same-tx rows. `sum` and `avg` require numeric columns; `min` and `max` also
accept string and timestamp columns and bind a result with the aggregated
column's type. `set`/`values` may hold a flat Value expression — arithmetic
(`+ - * /`), numeric rounding (`round`/`floor`/`ceil`), or date
(`date_add`/`date_sub` with a duration token like `3d`, optionally
`business_days` against a named `calendar` ObjectType — which MUST declare
`date` and `org_id` properties, see Recipe 6). `date_part` supports
`iso_weekday` · `year` · `month` · `day` and returns a number. Add
`tz: "org"` to use the org's nullable `_orgs.default_timezone`, or an IANA
literal such as `tz: "Europe/Copenhagen"` for a per-expression override;
explicit literals are checked against the platform tz database at declaration.
Without `tz`, the historical naive-UTC behavior is byte-for-byte unchanged;
an unset org setting also falls back to UTC. In a timezone, `d`/`w` preserve
local wall-clock time while `s`/`m`/`h` remain elapsed durations, and
`business_days` walks local dates. Date parts use the requested zone's
wall-clock fields when `tz` is present, otherwise UTC. Date-only strings
`YYYY-MM-DD` denote that calendar date, so their weekday is exact.

DST uses the IANA database: a nonexistent spring-forward wall time moves
forward by the transition gap (compatible disambiguation), while an autumn-back
overlap chooses the earlier instant. These rules are explicit and apply to
both normal and business-day arithmetic.

Numeric rounding uses this exact model: shift `left` by `10^decimals`; `round`
applies HALF-UP on the shifted value with half-away-from-zero ties (a `.5`
tie moves away from zero for both positive and negative values, so
`round(-2.5, 0)` is `-3`); `floor`/`ceil` apply `Math.floor`/`Math.ceil` to
the shifted value; then unshift by the same factor. `decimals` is an optional
integer literal from 0 through 6 (default 0). A `Number.EPSILON` correction
keeps decimal ties stable: `round(66.375, 2)` is exactly `66.38`,
`round(1.005, 2)` is `1.01`, and `round(2.675, 2)` is `2.68`.

```json
{ "op": "round", "left": "$price.amount", "decimals": 2 }
```

Expressions are flat in v1 (no nested
operands). All recipes, plus the `on_fail`
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

A second built-in surface is **the outbox queue**: `outbox.inspect` (a
governed read over `_outbox` — health counts plus a bounded row list, never
the raw `payload`) and `outbox.requeue` (put a deadlettered or
lease-abandoned effect row back on the queue). Both require `manage`
authority (owner/admin only). Full detail — the exact response shapes, the
`last_error` redaction + 240-char cap, and the `claim_live` /
`already_delivered` / `not_an_effect` refusals — is in
[resources/outbox-actions.md](resources/outbox-actions.md).

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
- [resources/outbox-actions.md](resources/outbox-actions.md) — the built-in `outbox.inspect` / `outbox.requeue` operator surface: queue health + row inspection, `last_error` redaction, and the requeue refusal/no-op reasons.

## Source-of-truth references

- Base skill: `canon skill --name canonify instructions` — the platform tour + runtime frame.
- `packages/canon/src/handler-dsl/schema.ts` — the `HandlerDsl` effect/Schema.
- `packages/canon/src/types.ts` — property/link/governance types.
- `packages/canon/src/actions/actions/declare.ts` — `actions.declare` input.
- `packages/canon/src/actions/catalog/refine-object-type.ts` — `catalog.refine_object_type` input + validator.
