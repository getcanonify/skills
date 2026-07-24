# Action handler patterns

The handler is the body of an ActionType: a **linear sequence of typed CRUD
steps over ObjectTypes**, glued by the `$ref` grammar, optionally terminated by
a `returns` block and an `on_fail` compensator. Every example here is a
complete `handler` object that decodes against the live `HandlerDsl`
effect/Schema (`packages/canon/src/handler-dsl/schema.ts`); the predicate
shape is from `packages/canon/src/handler-dsl/predicate.ts`. Wrap any of these
in the [declaration envelope](actions-input-governance.md) to register them.

## The ref grammar (recap)

| ref | resolves to |
|---|---|
| `$input.<field>` | a field on the schema-decoded action input |
| `$<step>.<field>` | a field on a previously bound `read`/`insert` step |
| `$now` | the runtime clock (ISO-8601 UTC) |
| `$principal.id` / `.org_id` / `.kind` | the caller identity |
| `$actor.<attr>` | a server-set external-principal attribute (write-side twin of the row-filter `:actor.<col>`); addresses an `actor_attr`-target row. Never reads the payload; fails closed off-external or on a missing attribute |

A value-position string starting with `$` MUST be a valid ref (lowercase-snake
head + dot segments) — `$Input.x` (capital) is rejected at decode. A literal
string that needs a leading `$` is escaped `$$`.

## The step kinds

`read` · `read_optional` · `read_aggregate` · `insert` · `update` · `delete` ·
`assert`. Each step is discriminated by its `op`. Every step may carry an
optional `when` predicate guard (except `assert`, where the predicate is the
mandatory `when`).

---

## Recipe 1 — read-then-update by id

The everyday "load a row, mutate it" handler. `read` always binds a name with
`as`; `update` takes `where` + `set`.

```json
{
  "steps": [
    { "op": "read", "object": "task", "where": { "id": "$input.task_id" }, "as": "task" },
    { "op": "update", "object": "task", "where": { "id": "$input.task_id" }, "set": { "title": "$input.title", "updated_at": "$now" } }
  ],
  "returns": { "task_id": "$input.task_id" }
}
```

## Recipe 2 — multi-table atomic insert with returns

Insert a parent, insert a child referencing the parent's freshly-stamped id,
then return the new ids. `insert` binds a name with `as` (optional, but you
need it to reference the row later). The executor stamps `id`, `created_at`,
`updated_at` automatically.

```json
{
  "steps": [
    { "op": "read", "object": "lead", "where": { "id": "$input.lead_id" }, "as": "lead" },
    { "op": "insert", "object": "customer", "values": { "name": "$lead.name", "email": "$lead.email", "org_id": "$principal.org_id" }, "as": "customer" },
    { "op": "insert", "object": "contact", "values": { "customer_id": "$customer.id", "name": "$lead.name", "email": "$lead.email", "role": "primary" }, "as": "contact" },
    { "op": "update", "object": "lead", "where": { "id": "$input.lead_id" }, "set": { "customer_id": "$customer.id", "converted_at": "$now" } }
  ],
  "returns": {
    "lead_id": "$input.lead_id",
    "customer_id": "$customer.id",
    "contact_id": "$contact.id"
  }
}
```

Everything runs in one transaction — any step failing rolls the whole thing
back.

## Recipe 3 — find-or-create with `read_optional`

`read_optional` has the same shape as `read` (mandatory `as`) but binds `null`
instead of failing `NOT_FOUND` when zero rows match. Combine with a `when`
guard that probes `present`/`absent` to express find-or-create.

```json
{
  "steps": [
    { "op": "read_optional", "object": "customer", "where": { "email": "$input.email" }, "as": "existing" },
    { "op": "insert", "object": "customer", "values": { "name": "$input.name", "email": "$input.email", "org_id": "$principal.org_id" }, "as": "created", "when": { "ref": "$existing.id", "absent": true } }
  ]
}
```

The insert only runs when the optional read bound nothing (`$existing.id` is
absent).

## Recipe 4 — `assert` precondition

`assert` has no `object`/`where`. It evaluates its (mandatory) `when` predicate
and, on FALSE, aborts the whole transaction with a typed domain error carrying
your verbatim `else.message`. `else.code` is restricted to `VALIDATION` or
`FORBIDDEN` — you cannot fabricate a server-fault code from a declarative
guard.

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "assert", "when": { "ref": "$order.status", "eq": "open" }, "else": { "code": "VALIDATION", "message": "order must be open to add items" } },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "item_count": { "op": "+", "left": "$order.item_count", "right": 1 } } }
  ]
}
```

A predicate carries a `ref` plus **exactly one** operator:
`eq` · `neq` · `gt` · `gte` · `lt` · `lte` · `in` · `present` · `absent`.
Two operators, zero operators, or an unknown operator all fail the decode.
There is no `and`/`or`/`not` — chain multiple `assert` steps for conjunction.

**The operand resolves refs too.** The value side of a comparison (`eq`/`neq`/
`gt`/`gte`/`lt`/`lte`, and each `in` member) may itself be a `$ref`/`$now`
string — it resolves through the **same** grammar as the `ref` on the left. The
canonical use is a **cut-off / date guard**, `{ "ref": "$input.day_id", "lt":
"$now" }`, which compares against the resolved clock instant, not the literal
characters `"$now"`:

```json
{
  "steps": [
    { "op": "assert", "when": { "ref": "$input.day_id", "lt": "$now" }, "else": { "code": "VALIDATION", "message": "the cut-off for this day has passed" } },
    { "op": "insert", "object": "order", "values": { "day_id": "$input.day_id", "placed_at": "$now" }, "as": "order" }
  ],
  "returns": { "order_id": "$order.id" }
}
```

A literal string in the value position that does **not** match the ref grammar
(`{ "eq": "churned" }`, `{ "in": ["active", "trialing"] }`) is compared verbatim,
unchanged — only a valid `$`-prefixed ref is resolved.

## Recipe 5 — `read_aggregate`

A scalar aggregate over an ObjectType, bindable into a later write. Exactly one
aggregator per step: `count: true`, or `sum`/`avg`/`min`/`max` carrying the
column name. `as` is mandatory; the result binds under `$<as>.<op>`.

```json
{
  "steps": [
    { "op": "read_aggregate", "object": "task", "where": { "customer_id": "$input.customer_id" }, "count": true, "as": "tally" },
    { "op": "update", "object": "customer", "where": { "id": "$input.customer_id" }, "set": { "open_task_count": "$tally.count" } }
  ]
}
```

For a column aggregate:

```json
{
  "steps": [
    { "op": "read_aggregate", "object": "order", "where": { "customer_id": "$input.customer_id" }, "sum": "amount_cents", "as": "revenue" },
    { "op": "update", "object": "customer", "where": { "id": "$input.customer_id" }, "set": { "lifetime_cents": "$revenue.sum" } }
  ]
}
```

Because `read_aggregate` rides the typed-write path, it sees rows written
*earlier in the same transaction*.

## Recipe 6 — arithmetic & date expressions

A `set`/`values` field may hold a flat binary expression instead of a value.
Arithmetic ops (`+ - * /`) take two operands that must resolve to numbers.

```json
{
  "steps": [
    { "op": "read", "object": "counter", "where": { "id": "$input.counter_id" }, "as": "counter" },
    { "op": "update", "object": "counter", "where": { "id": "$input.counter_id" }, "set": { "value": { "op": "+", "left": "$counter.value", "right": 1 } } }
  ]
}
```

Date arithmetic uses `date_add`/`date_sub` with a duration token
(`30s`/`15m`/`1h`/`1d`/`1w`; `m` = minutes). `business_days: true` counts only
working days and reads skip-dates from a named `calendar` ObjectType (only
`d`/`w` durations are valid under `business_days`):

```json
{
  "steps": [
    { "op": "insert", "object": "task", "values": { "title": "$input.title", "due_at": { "op": "date_add", "left": "$now", "right": "3d", "business_days": true, "calendar": "holiday" } } }
  ]
}
```

Expressions are **flat** in v1 — an operand cannot itself be an expression.
Compose deeper math with sequential update steps.

## Recipe 7 — `on_fail` compensator

`on_fail` rides the handler surface and names an ActionType to enqueue (on the
outbox) when an `assert` or the FSM transition guard rejects. Its `inputs` may
reference `$input`/`$principal`/`$now` but **not** a `$<step>.*` binding (those
don't survive the rolled-back tx).

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "assert", "when": { "ref": "$order.status", "neq": "cancelled" }, "else": { "code": "VALIDATION", "message": "cannot ship a cancelled order" } },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "shipped_at": "$now" } }
  ],
  "on_fail": {
    "action": "alert.raise",
    "inputs": { "order_id": "$input.order_id", "raised_at": "$now" }
  }
}
```

## Auto-CRUD on a compound root — the generated verbs change

The patterns above are for **custom** handlers you author. Auto-CRUD also
generates default actions per ObjectType, and for a plain (non-compound) type
that set is exactly five: `<type>.create`, `<type>.read`, `<type>.update`,
`<type>.delete`, `<type>.list` (each suppressible via `ObjectType.defaults`). The
moment a type declares a `part_of` link, the generated set grows and three of the
verbs gain compound behaviour (ADR 0002 — verified against
`packages/canon/src/auto-crud/crud-generator.ts`):

- **`<root>.create` accepts nested children.** When the root has `part_of` links,
  its create input grows an optional slot per link (`{ ...header, "lines":
  [ ...children ] }` — an array for `cardinality:"many"`, a single object for
  `"one"`). The whole cluster is inserted **atomically and recursively** in one
  transaction — a 3-level `part_of` tree inserts in one walk; any failure rolls
  the whole cluster back. Each child's FK (`via`) is **auto-bound** to the new
  root id; a caller who *also* supplies that FK is rejected (`VALIDATION`). A flat
  create (no children supplied) is byte-identical to the pre-compound behaviour.

- **A new `<root>.replace` action is generated — compound roots only.** No plain
  type gets a `.replace`. It is a PUT-style **whole-compound** write: submit
  `{ id, ...header, "lines": [...] }` and it reconciles children against current
  state in one transaction — children with an `id` are **updated**, children
  without an `id` are **inserted** (FK auto-bound), and current children **absent
  from the submitted list are deleted by omission** (subtree-cascaded; an omitted
  row's non-`part_of` inbound links still honour their `on_delete`). The sharp
  edge: an **absent** key for a declared `part_of` link is rejected — supply `[]`
  (many) or `null` (one) to delete all, never omit the key. No cross-compound
  adoption: an `id` that belongs to another root is a `VALIDATION` error.

- **Part-level `<part>.create` / `<part>.update` / `<part>.delete` gain a freeze
  gate + version anchor.** When a type is itself a *part* of some compound, its
  flat create/update/delete is a compound write: in the same transaction it
  runs the root's **version anchor** (a check-and-bump that serialises sibling-part
  edits through the root) and the **`parts_frozen_when` freeze gate** (a frozen
  root rejects the part write with `CONFLICT` / `details.reason === "FROZEN"`).
  Each carries an optional `expected_root_version` input: supplied, a stale root
  version is a typed `CONFLICT`; omitted, the write still bumps and gates but
  skips the staleness compare-and-set.

Reading a compound back: `canon objects get <type> <id> --expand` returns the
root with its `part_of` children nested (the `getExpanded` read; over REST it is
`GET /v1/objects/<type>/<id>?expand=1`). Declared `rollups` are merged onto the
row here, in `get`, and in `list`. A runtime adapter that predates expand support
returns a `VALIDATION` error rather than silently dropping the flag.

## Gotchas the typecheck pass rejects

- **Writing an FSM-bound state column** in a transition action →
  `VALIDATION/direct_state_write`. Let the runtime auto-write it from the
  `transition.to` marker.
- **A `$ref` to an unbound step** (`$foo.x` where no step is named `foo`).
- **A malformed ref** (`$Input.id` with a capital, `$.x`, `$1abc`) — caught at
  decode by the `Ref` pattern.
- **An arithmetic operand that isn't numeric**, or a `date_add` operand that
  isn't a date / a non-duration `right` token.
- **An empty handler** — `steps` must be a non-empty array
  (`"handler must declare at least one step"`).
- **`read_aggregate` with zero or multiple aggregators** —
  `"read_aggregate requires EXACTLY ONE aggregator"`.

## Security gotcha: writing scope columns on a `target`-declared action

If your ActionType declares a write `target` (ADR-0005 — a derived-target
`actor_attr` or by-reference `input_key` scoping for an external write, e.g.
`guest.set_standing_line`), the external write gate scope-checks the
*resolved target row*, not your handler's `insert`/`update` `values`/`set`.
Concretely: **never bind a scope column (`customer_id`, an org-scoping
column, etc.) from `$input.<field>`** in a step that writes it — a client can
supply any value there and the gate will not catch it, because a declared
`target` bypasses the payload-candidate check entirely. Always derive scope
columns from `$principal.*` / the server-resolved target row (e.g. read the
target first with a `read` step keyed by the target id, then reference
`$<that-step>.customer_id` in your write), never from raw `$input.*`. See
[ADR-0005 "Authoring guidance / hazards"](../../../adr/0005-external-write-gate-target-scoping.md#authoring-guidance--hazards)
for the full writeup.

## See also

- [actions-input-governance.md](actions-input-governance.md) — the declaration envelope.
- Base skill: `canon skill --name canonify resource handler-dsl-cookbook.md` and the error-code reference `resource error-codes.md`.
