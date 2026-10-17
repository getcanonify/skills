# The handler DSL

A handler is an **ordered list of typed steps over ObjectTypes**, glued by a
reference grammar, with an optional `returns` block and `on_fail`
compensator. It is deliberately small and closed — type-checked at declare
time against the live registry, not a programming language: no `if`/`else`
(two steps with opposite `when`s is the idiom), no string concatenation, no
function calls, no SQL (SQL lives in derived properties and the read-only
`sql_read`). Every step runs in one transaction with the runtime's guard,
audit and outbox around it; any failure rolls the whole action back. The
envelope around the handler is [actions-input-governance.md](actions-input-governance.md).

## Refs

| Ref | Resolves to |
|---|---|
| `$input.<field>` | a field of the decoded action input |
| `$<step>.<field>` | a field of a row a prior step bound with `as` |
| `$<read_many_step>` | a row set — usable as `for_each.in`; `$set.count` is its size; `$set.<property>` only as a direct `where.in` operand |
| `$now` | the runtime clock, ISO-8601 UTC |
| `$principal.id` / `.org_id` / `.kind` | the caller |
| `$actor.<attr>` | a server-set attribute of an external principal (never the payload); addresses an `actor_attr` target row; fails closed (`VALIDATION`) for any other principal or a missing attribute |

A value-position string starting with `$` must be a valid ref
(`/^\$(input|principal|now|[a-z_][a-z0-9_]*)(\.[a-z_][a-z0-9_]*)*$/` —
`$Input.x` is rejected at decode); write a literal leading `$` as `$$`. Other
values are JSON literals.

## Steps

| `op` | Does | `as` |
|---|---|---|
| `read` | one row by `where`; `NOT_FOUND` if none | required |
| `read_optional` | one row or `null`; takes `order_by`, `limit` | required |
| `read_many` | a row set in ascending `id` order (`order_by`, `limit`), at most 500 | required |
| `read_aggregate` | exactly one of `count: true` / `sum` / `avg` / `min` / `max` over a `where`, optional `group_by`; binds `$<as>.<op>` | required |
| `insert` | a row from `values`; `id`, `created_at`, `updated_at` stamped | optional |
| `update` | every row matching `where` gets `set`; `updated_at` stamped | — |
| `delete` | rows matching `where`, honouring links' `on_delete`; zero matches is a no-op | — |
| `assert` | aborts with `else: { code: VALIDATION \| FORBIDDEN, message, field? }` unless `when` holds | — |
| `transition` | names a row as a subject of the action's own transition marker; writes nothing itself | — |
| `call` | runs a declared fragment with inputs bound by `with` | optional |
| `for_each` | runs nested steps once per element of a `read_many` set (`in`); no nested `for_each` | the element |

Every step takes an optional `when` guard (mandatory on `assert`). An `update`
is set-wide — only zero matches is `NOT_FOUND` — so key a single-row write by
something identifying, and `read` first when a missing id must be a
`NOT_FOUND` rather than a silent no-op.

## The canonical example — `lead.convert`

A four-step atomic multi-table write under a transition marker:

```json
{
  "name": "lead.convert",
  "description": "Convert a qualified lead into a Customer + primary Contact.",
  "object_type": "lead",
  "input": { "lead_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": { "state_machine": "Lead.status", "from": ["qualified"], "to": "converted" },
  "handler": {
    "steps": [
      { "op": "read", "object": "lead", "where": { "id": "$input.lead_id" }, "as": "lead" },
      { "op": "insert", "object": "customer", "values": { "name": "$lead.name", "email": "$lead.email", "org_id": "$principal.org_id" }, "as": "customer" },
      { "op": "insert", "object": "contact", "values": { "customer_id": "$customer.id", "name": "$lead.name", "email": "$lead.email", "role": "primary" }, "as": "contact" },
      { "op": "update", "object": "lead", "where": { "id": "$input.lead_id" }, "set": { "customer_id": "$customer.id", "converted_at": "$now" } }
    ],
    "returns": { "lead_id": "$input.lead_id", "customer_id": "$customer.id", "contact_id": "$contact.id" }
  }
}
```

At invoke, inside one transaction: idempotency and governance checks, input
decode, the guard (`lead.status` must be `qualified`), the four steps (the
inserts stamp `id` from each type's `idPrefix`, `created_at`, `updated_at`),
then the runtime writes `status = 'converted'` (the handler never does), enqueues
the machine's `on_enter.converted` hooks to the outbox, and writes the audit
row. The envelope is `{ data: { lead_id, customer_id, contact_id },
audit_event_id }`. Writing `customer.org_id` needs that hidden column surfaced
on `customer` first (`hidden: false`, [object-property-types.md](object-property-types.md)).

A one-step transition (`lead.contact`) is the same declaration with
`from: ["new"]`, `to: "contacted"` and one `update` stamping `contacted_at`.
With nothing to stamp, use one `read` — a handler needs at least one step.

## Conditions — `when`, `assert`, predicates

A predicate is a `ref` plus **exactly one** operator: `eq` · `neq` · `gt` ·
`gte` · `lt` · `lte` · `in` · `present` · `absent` · `contains` · `has_any` ·
`has_all`. The operand may be a literal, a ref or an expression: the cut-off
guard `{ "ref": "$input.day_id", "lt": "$now" }` compares against the resolved
clock. A literal that is not a valid ref (`{ "eq": "churned" }`) compares
verbatim.

In a `when` or `assert`, `any_of` is a flat disjunction of full predicates:
each member needs its own `ref`, and `any_of` members may carry different refs.
For example, “no guest named OR named guest belongs to this customer” is
`{ "any_of": [{ "ref": "$input.guest_id", "absent": true }, { "ref": "$guest.id", "present": true }] }`.

`all_of` is the conjunction, **guard-only** (never in a `where`, whose columns
already AND). Its members are leaves or `any_of` groups — conjunctive normal
form one level deep: `all_of` in `all_of` is `predicate_all_of_flat`, `all_of`
in `any_of` is `predicate_any_of_flat`, and there is no `not`. Prefer it to
chained asserts when the conditions are one decision, and always to count
arithmetic (`eq: { "op": "-", "left": 2, "right": "$locked.count" }` passes
silently on one miscount).

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "assert",
      "when": { "all_of": [
        { "ref": "$order.locked", "eq": false },
        { "any_of": [{ "ref": "$order.status", "eq": "open" }, { "ref": "$order.status", "eq": "draft" }] }
      ] },
      "else": { "code": "VALIDATION", "message": "order must be open to add items", "field": "order_id" } },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "item_count": { "op": "+", "left": "$order.item_count", "right": 1 } } }
  ]
}
```

`assert` aborts with your verbatim message; `else.field` adds
`failed_precondition: { field }` to the envelope so a form can pin the message
to that field.

`contains` is a predicate operator for JSON-array membership. The property must
be `json` or `set<enum>` (`contains_requires_json` otherwise) and the operand
must resolve to a scalar — `string`, `number`, `boolean`, `enum`, `money`,
`date` or `timestamp` (`contains_operand_scalar` for JSON or null). Membership
is type-strict (`[1, 2]` does not contain `"1"`); a NULL cell never matches;
malformed JSON is `VALIDATION`. For example:
`when: { "ref": "$guest.tags", "contains": "vip" }`.

`has_any` / `has_all` work on a `set<enum>` only
(`set_operator_requires_set_enum`): an array of literals or refs (never
expressions) that the stored set shares one of / contains all of.
`when: { "ref": "$item.dietary_variant", "has_all": ["vegan", "gluten_free"] }`.
A whole set is also an equality operand — `where: { "dietary_variant":
"$input.dietary_variant" }` matches exactly those members in any order.

### `where`

Each `where` key takes a scalar value, one operator object (`gt` · `gte` · `lt`
· `lte` · `neq` · `in` · `present` · `absent` · `contains` · `has_any` ·
`has_all`, exactly one key), or one flat `any_of` of those. A literal `in` array
is non-empty; a ref `in` operand must be a property of a prior `read_many` set
and compiles to a bound `IN` list (`1=0` when empty). `contains`/`has_any`/
`has_all` compile to `EXISTS`/`COUNT(DISTINCT …)` over `json_each`; an empty
`has_any` is `1=0`, an empty `has_all` `1=1`. A filter is capped (1000 members
per array, 10000 bound values, limited nesting): `where_too_large` /
`where_too_deep`. Temporal types take the `$as_of` key.

```json
{
  "steps": [
    { "op": "read_many", "object": "standing_order",
      "where": { "location_id": "$input.location_id",
                 "weekdays": { "contains": { "op": "date_part", "part": "iso_weekday", "left": "$input.delivery_date" } } },
      "as": "orders" }
  ]
}
```

Declare such a `weekdays` column `set<enum>` (or `json`), never `string`.

### Nullable bindings and narrowing

A `when`-guarded step binds its `as` to `null` when skipped, and
`read_optional` binds `null` on no row. A strict write position — `values`,
`set`, a bare `where` equality — may take such a binding only into a nullable
property; otherwise `nullable_binding_into_non_nullable` names the alias, its
guard and the target. Predicates stay null-tolerant. Narrowing is
flow-sensitive: `present: true` (or `neq: null`) on a binding narrows it for
that step's own maps; the same guard on a `for_each` narrows the whole body; an
`assert` narrows every following step in the block; an `all_of` narrows what
its top-level members prove; a leaf inside `any_of` proves nothing. In
`read_optional`/`read_many`/`read_aggregate` filters a null binding becomes
`IS NULL` (yielding null, an empty set or zero).

Find-or-create:

```json
{
  "steps": [
    { "op": "read_optional", "object": "customer", "where": { "email": "$input.email" }, "as": "existing" },
    { "op": "insert", "object": "customer", "values": { "name": "$input.name", "email": "$input.email" }, "as": "created", "when": { "ref": "$existing.id", "absent": true } }
  ]
}
```

## Value expressions

Any value — a `values`/`set` field, a `where` or predicate operand, a `with`
binding — may be an expression node:

| Node | Shape |
|---|---|
| arithmetic | `{ "op": "+" \| "-" \| "*" \| "/", "left", "right" }` (numeric operands; `money` counts) |
| rounding | `{ "op": "round" \| "floor" \| "ceil", "left", "decimals"? }` |
| date arithmetic | `{ "op": "date_add" \| "date_sub", "left", "right": "<duration>", "tz"?, "business_days"?, "calendar"? }` |
| date part | `{ "op": "date_part", "part": "iso_weekday" \| "year" \| "month" \| "day", "left", "tz"? }` |

Operands nest: any `left`/`right` may itself be an expression, nested at most 5
levels (`value_expr_too_deep` beyond). A `date_add`/`date_sub` `right` stays a
literal duration and `has_any`/`has_all` members stay flat.

```json
{
  "op": "round", "decimals": 2,
  "left": { "op": "/", "right": 100,
    "left": { "op": "*", "left": "$price.list_price",
      "right": { "op": "-", "left": 100, "right": "$input.discount_pct" } } }
}
```

`73.75 × (100 − 10) / 100 = 66.375` freezes as `66.38` (depth 4). Rounding
shifts by `10^decimals` (integer 0–6, default 0), then `round` is half-up away
from zero (`round(-2.5)` = `-3`), `floor`/`ceil` are the math functions, with an
epsilon correction so `round(1.005, 2)` = `1.01`. Arithmetic over `money` stays
in minor units and a money column refuses a fractional result — round where
your rounding policy says.

**Dates.** Durations are `30s` · `15m` · `1h` · `1d` · `1w` (`m` = minutes;
`Nw` = 7N days, same weekday). Without `tz`, arithmetic is UTC milliseconds and
`date_part` reads UTC fields. `tz: "org"` uses the org's timezone (unset → UTC),
`tz: "Europe/Copenhagen"` any IANA zone (checked at declare): `d`/`w` then keep
the local wall-clock time, `s`/`m`/`h` stay elapsed time. A wall time in a
spring-forward gap moves forward by the gap (02:30 → 03:30); in an autumn-back
overlap the earlier instant wins. `business_days: true` (only `d`/`w`) skips
local Saturday/Sunday and the dates of a `calendar` ObjectType, and then
`Nw` = 5N business days. That calendar type must declare mapped `date` and
`org_id` properties (rows match the caller's org exactly; there is no global
calendar) — apply and validate refuse a handler whose calendar lacks either.
`date_part` on a `YYYY-MM-DD` string is that calendar date; its `left` must be
a `timestamp` or `string` (`date_part_left_not_date`); `hour` is not a part.

```json
{
  "steps": [
    { "op": "insert", "object": "task", "values": { "title": "$input.title", "due_at": { "op": "date_add", "left": "$now", "right": "3d", "business_days": true, "calendar": "holiday" } } }
  ]
}
```

## Aggregates

`read_aggregate` takes one aggregator; `sum`/`avg` need a `number` or `money`
column, `min`/`max` also take string, timestamp and date columns (the result
keeps the column's type). It sees rows written earlier in the same transaction.

```json
{
  "steps": [
    { "op": "read_aggregate", "object": "order", "where": { "customer_id": "$input.customer_id" }, "sum": "amount_cents", "as": "revenue" },
    { "op": "update", "object": "customer", "where": { "id": "$input.customer_id" }, "set": { "lifetime_cents": "$revenue.sum" } }
  ]
}
```

Minting a sequential human code (`J-1042`) with `max` + 1 and a derived
property is a pattern in
[canonify-app-authoring › patterns.md](../../canonify-app-authoring/resources/patterns.md).

## Sets — `read_many` and `for_each`

`read_many` binds at most **500** rows. Without `limit` it probes one past the
cap, and more matches fail the whole action with `VALIDATION`
`details: { code: "row_cap_exceeded", step, object, as, cap: 500 }` — never a
silent truncation; an explicit `limit` (1–500) takes a bounded subset. Its
`where` must not be empty (`read_many_where_empty`). `order_by` is one
`{ property, direction }` over a number, money, string, timestamp or date
property (a derived one too); on `read_optional` it picks the returned row.
`$set.count` (0–500) reports the rows bound — name it `rows_iterated`, it is not
a write count. A child collection is reached without nesting loops through a
`where.in` projection:

```json
{
  "steps": [
    { "op": "read_many", "object": "order", "where": { "delivery_date": "$input.delivery_date" }, "as": "orders" },
    { "op": "for_each", "in": "$orders", "as": "order", "steps": [
      { "op": "insert", "object": "order_note", "values": { "order_id": "$order.id", "body": "$input.note" } }
    ] },
    { "op": "update", "object": "order_line", "where": { "order_id": { "in": "$orders.id" } }, "set": { "locked": true } }
  ],
  "returns": { "rows_iterated": "$orders.count" }
}
```

An empty set is a no-op. The loop alias may not shadow a binding or the heads
`input`, `principal`, `now`, `actor` (`for_each_alias_shadow`); loop-local
bindings end with the loop. A set used as a row (`$orders.id` outside a
`where.in`, bare `$orders` outside `for_each.in`) is
`set_binding_not_addressable`.

**Set transitions.** A `transition` step inside a `for_each` names each row the
action's marker moves; the runtime guards, writes and fires hooks per row after
the handler, atomically — rules and example in
[state-machines.md](state-machines.md). Creating a record that starts past the
initial state (insert + `transition` in one action) is a pattern in
[canonify-app-authoring › patterns.md](../../canonify-app-authoring/resources/patterns.md).

## Effective-dated reads

For a `temporal` type, filter with `$as_of`. For hand-modelled periods, use a
range on the start and a flat `any_of` for an open (NULL) or later end:

```json
{
  "steps": [
    { "op": "read_optional", "object": "price",
      "where": { "product_id": "$input.product_id",
                 "valid_from": { "lte": "$input.as_of" },
                 "valid_to": { "any_of": [{ "absent": true }, { "gt": "$input.as_of" }] } },
      "order_by": { "property": "valid_from", "direction": "desc" }, "limit": 1, "as": "price" }
  ]
}
```

Periods are half-open `[valid_from, valid_to)`. An overlap check before an
insert is the same read against the new start, followed by `assert` on
`absent`. Never compare against `null` (`{ "lt": null }`) — SQL NULL never
satisfies it; use `absent`.

## Fragments — `call`

A block several handlers share (a price lookup, a guard) is declared once with
`fragments.declare` (a bundle's `fragments/<name>.json`, applied before
`actions/`) and run with `call`. A fragment is straight-line — reads, writes,
asserts; no `for_each`, `transition` or `call` — runs in the caller's
transaction as the caller, sees only its own `$input` (from `with`), `$now` and
`$principal`, and exposes its `returns` as `$<as>.<field>`.

```json
{
  "name": "price_lookup",
  "description": "The customer price in effect on the date, else the list price.",
  "input": {
    "customer_id": { "type": "string" },
    "product_id": { "type": "string" },
    "delivery_date": { "type": "string" }
  },
  "steps": [
    { "op": "read_optional", "object": "price", "as": "cust",
      "where": { "product_id": "$input.product_id", "customer_id": "$input.customer_id",
                 "valid_from": { "lte": "$input.delivery_date" },
                 "valid_to": { "any_of": [{ "absent": true }, { "gt": "$input.delivery_date" }] } } },
    { "op": "read_optional", "object": "price", "as": "list",
      "where": { "product_id": "$input.product_id", "customer_id": { "absent": true },
                 "valid_from": { "lte": "$input.delivery_date" },
                 "valid_to": { "any_of": [{ "absent": true }, { "gt": "$input.delivery_date" }] } } }
  ],
  "returns": { "cust_amount": "$cust.amount_minor", "list_amount": "$list.amount_minor", "list_id": "$list.id" }
}
```

```json
{ "op": "call", "fragment": "price_lookup", "as": "price",
  "with": { "customer_id": "$line.customer_id", "product_id": "$line.product_id",
            "delivery_date": "$input.delivery_date" } }
```

- `with` binds every required input (`call_input_missing`), no undeclared key
  (`call_input_unknown`); values are type-checked. Reading a field the fragment
  does not return is `call_returns_no_field`; an undeclared fragment is
  `unknown_fragment`.
- A nullable binding may flow only into an `optional`/`nullable` input; a field
  returned from a `read_optional` row is nullable at the call site too.
- A skipped call binds `null`. `call` is legal at top level and in a
  `for_each` body.
- A fragment may not write a machine-bound property (`direct_state_write`).
- Re-declaring needs `expectedHash` (from `catalog.export`) or `force`; a change
  that breaks a caller is `fragment_caller_broken`. Apply refuses to prune a
  fragment a surviving action calls (`unknown_fragment` / `fragment_in_use`).

## `on_fail`

Names an action to enqueue on the outbox when an `assert` or the transition
guard rejects. Its `inputs` may use `$input`, `$principal` and `$now`, never a
step binding (the transaction rolled back).

```json
{
  "steps": [
    { "op": "read", "object": "order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "update", "object": "order", "where": { "id": "$input.order_id" }, "set": { "shipped_at": "$now" } }
  ],
  "on_fail": { "action": "alert.raise", "inputs": { "order_id": "$input.order_id", "raised_at": "$now" } }
}
```

## What the type-check rejects

All are `VALIDATION` at declare (`typecheck_failed` unless noted), citing the
step path (`steps[6].steps[3].where.valid_to`) and a label such as
`for_each $orders › update order_line`.

| Code / message | Cause |
|---|---|
| `unknown object: leed` · `unknown ref: $customer.id` · `lead has no property "foobar"` | typo'd type, unbound step, unknown field |
| ref pattern (decode) | `$Input.id`, `$.x`, `$1abc` |
| `handler must declare at least one step` | empty `steps` |
| `direct_state_write` | writing a machine-bound property |
| `invalid_transition_step` | a `transition` step without a marker, or on another type |
| `read_aggregate requires EXACTLY ONE aggregator` | zero or several aggregators |
| `for_each_nesting` · `for_each_in_not_set` · `for_each_alias_shadow` · `for_each_body_scope` | loop misuse |
| `set_binding_not_addressable` · `where_in_set_binding_required` · `read_many_where_empty` · `read_limit_invalid` | set misuse; `limit` must be a positive integer (≤ 500 on `read_many`) |
| `read_order_by_not_orderable` · `where_op_not_orderable` · `predicate_not_orderable` | ordering a non-orderable property or ref |
| `where_op_exactly_one` · `where_op_type_mismatch` · `where_absent_on_non_nullable` · `where_any_of_flat` | `where` operator misuse |
| `predicate_operand_unresolved` · `predicate_operand_type_mismatch` · `predicate_any_of_flat` · `predicate_all_of_flat` | predicate misuse |
| `contains_requires_json` · `contains_operand_scalar` · `set_operator_requires_set_enum` | membership misuse |
| `value_expr_too_deep` · non-numeric arithmetic · bad duration | expression misuse |
| `date_part_left_not_date` · unknown `part` (decode) | date part misuse |
| `nullable_binding_into_non_nullable` | a maybe-null binding into a required write |
| `unknown_fragment` · `call_input_missing` · `call_input_unknown` · `call_returns_no_field` | fragment misuse |
</content>
