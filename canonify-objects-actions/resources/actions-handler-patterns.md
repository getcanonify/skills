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
| `$<step>.<field>` | a field on a previously bound row-producing step (not a `read_many` set) |
| `$<read_many_step>` | a row set usable as `for_each.in`; `$set.<property>` is also allowed only as a direct `where.in` operand, and `$set.count` reports its size |
| `$now` | the runtime clock (ISO-8601 UTC) |
| `$principal.id` / `.org_id` / `.kind` | the caller identity |
| `$actor.<attr>` | a server-set external-principal attribute (write-side twin of the row-filter `:actor.<col>`); addresses an `actor_attr`-target row. Never reads the payload; fails closed off-external or on a missing attribute |

A value-position string starting with `$` MUST be a valid ref (lowercase-snake
head + dot segments) — `$Input.x` (capital) is rejected at decode. A literal
string that needs a leading `$` is escaped `$$`.

## The step kinds

`read` · `read_optional` · `read_many` · `read_aggregate` · `insert` · `update` ·
`delete` · `assert` · `for_each`. Each step is discriminated by its `op`. Every
step may carry an optional `when` predicate guard (except `assert`, where the
predicate is the mandatory `when`). `read_many` binds a row set and
`for_each` iterates it with a body containing the existing seven step kinds;
the body cannot nest another `for_each`.

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

`else` also accepts an OPTIONAL `field` naming the input param the
precondition is about (e.g. `"else": { "code": "VALIDATION", "message":
"CVR is required", "field": "cvr" }`). When declared, the rejection envelope
carries `failed_precondition: { "field": "cvr" }`, so a rendering client can
attach your message to that form field instead of a top-level banner. Without
`field` the envelope is unchanged from the shape above.

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
`eq` · `neq` · `gt` · `gte` · `lt` · `lte` · `in` · `present` · `absent` · `contains`.
Two operators, zero operators, or an unknown operator all fail the decode.
Chain multiple `assert` steps for conjunction. In a `when` or `assert`,
`any_of` is a flat disjunction of full predicates: each member needs its own
`ref`, and `any_of` members may carry different refs. For example, “no guest
named OR named guest belongs to this customer” is
`{ "any_of": [{ "ref": "$input.guest_id", "absent": true }, { "ref": "$guest.id", "present": true }] }`.
Predicate `any_of` is flat: its members are leaves and cannot contain another
`any_of`.

`contains` is a predicate operator for JSON-array membership. For example,
`when: { "ref": "$guest.tags", "contains": "vip" }` checks a declared JSON
array property.

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

The one set-valued exception is a direct `where.in` operand such as
`{ "in": "$orders.id" }`: it projects that property from the bounded
`read_many` rows into bound parameters. A set property is still rejected in
every other value or ref position.

## Recipe 5 — `read_aggregate`

A scalar aggregate over an ObjectType, bindable into a later write. Exactly one
aggregator per step: `count: true`, or `sum`/`avg`/`min`/`max` carrying the
column name. `as` is mandatory; the result binds under `$<as>.<op>`.

`sum` and `avg` require numeric columns. `min` and `max` also accept string and
timestamp columns, and their bound value keeps the aggregated column's type.

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

### Monetary price freeze: discount, then round

For a monetary amount, calculate the percentage in separate flat expressions
and round only the final result. This example stores the intermediate
remaining percentage and unrounded amount in `discounted_price` inside the
same transaction, so no intermediate value is visible. The `price` object
must have numeric `list_price` and `discounted_price` properties.

```json
{
  "steps": [
    { "op": "read", "object": "price", "where": { "id": "$input.price_id" }, "as": "price" },
    { "op": "update", "object": "price", "where": { "id": "$input.price_id" }, "set": { "discounted_price": { "op": "-", "left": 100, "right": "$input.discount_pct" } } },
    { "op": "read", "object": "price", "where": { "id": "$input.price_id" }, "as": "remaining" },
    { "op": "update", "object": "price", "where": { "id": "$input.price_id" }, "set": { "discounted_price": { "op": "*", "left": "$price.list_price", "right": "$remaining.discounted_price" } } },
    { "op": "read", "object": "price", "where": { "id": "$input.price_id" }, "as": "scaled" },
    { "op": "update", "object": "price", "where": { "id": "$input.price_id" }, "set": { "discounted_price": { "op": "/", "left": "$scaled.discounted_price", "right": 100 } } },
    { "op": "read", "object": "price", "where": { "id": "$input.price_id" }, "as": "unrounded" },
    { "op": "update", "object": "price", "where": { "id": "$input.price_id" }, "set": { "discounted_price": { "op": "round", "left": "$unrounded.discounted_price", "decimals": 2 } } }
  ]
}
```

The arithmetic is `list_price × (100 − discount_pct) / 100`, followed by
`round(..., 2)`. For example, `73.75 × (100 − 10) / 100` is `66.375`, which
freezes as `66.38`. Re-read after each update because Value expressions are
flat and a later expression must reference the newest binding.

Date arithmetic uses `date_add`/`date_sub` with a duration token
(`30s`/`15m`/`1h`/`1d`/`1w`; `m` = minutes). In normal (non-`business_days`)
mode, `Nw` means N × 7 calendar days, landing on the same weekday.

An expression may add `tz: "org"` to use the org's nullable
`_orgs.default_timezone` setting, or `tz: "Europe/Copenhagen"` (any valid
IANA timezone literal) to override it for that expression. The declaration
checker validates explicit zones against the platform tz database. Omitting
`tz` is intentionally unchanged: arithmetic uses naive UTC milliseconds and
`date_part` reads UTC fields. An unset org timezone falls back to that same
UTC behavior. With a timezone, `d` and `w` arithmetic is calendar arithmetic
that preserves the local wall-clock time; `s`, `m`, and `h` remain elapsed
durations. `business_days` moves local calendar dates, skipping local
Saturday/Sunday and the declared calendar's dates.

Timezone conversion uses the IANA database at the target instant. A wall-clock
time in a spring-forward gap resolves with the **compatible** rule: move it
forward by the transition gap (for example, nonexistent 02:30 becomes 03:30).
When autumn-back creates two possible instants, choose the **earlier** instant
(the pre-transition offset). These rules apply to normal and business-day
arithmetic and are part of the handler contract.

`business_days: true` counts only working days and reads skip-dates from a
named `calendar` ObjectType (only `d`/`w` durations are valid under
`business_days`). **Under `business_days`, `Nw` means N WORKING WEEKS = N × 5
business days** — not N × 7 — so a clean week (no holidays) also lands on the
same weekday:

```json
{
  "steps": [
    { "op": "insert", "object": "task", "values": { "title": "$input.title", "due_at": { "op": "date_add", "left": "$now", "right": "3d", "business_days": true, "calendar": "holiday" } } }
  ]
}
```

**The `calendar` ObjectType contract is not optional.** It MUST declare two
mapped properties:

- `date` — the skip-date column. Each row is one closed day.
- `org_id` — the tenant-scope column. Reads use org-EQUALITY only
  (`{ org_id: principal.org_id }`); a NULL or "global" `org_id` row does
  **not** count for any org — there is no global-calendar fallback.

`canon catalog apply` and `canon catalog validate` both reject a handler
referencing a `calendar` ObjectType missing either property, naming the
calendar and the missing property. This runs at declare/apply time (not just
at invoke), so a bundle with an incomplete calendar never applies green only
to fail on first use.

### Date-part derivation

`date_part` is a flat Value expression that extracts one closed-set part from
an ISO-8601 date or timestamp: `iso_weekday` (1 = Monday through 7 = Sunday),
`year`, `month`, or `day`. It returns a number and can be used anywhere a
predicate operand accepts a Value, including the date-part half of a weekday
containment guard:

```json
{
  "steps": [
    {
      "op": "assert",
      "when": {
        "ref": "$input.weekday",
        "eq": { "op": "date_part", "part": "iso_weekday", "left": "$input.delivery_date" }
      },
      "else": { "code": "VALIDATION", "message": "not a delivery day" }
    }
  ]
}
```

Date parts are the UTC fields of the instant unless `tz` is supplied, in which
case they are the requested zone's wall-clock fields. A date-only string
`YYYY-MM-DD` denotes that calendar date, so its weekday is exact. Invalid dates
fail validation; `part: "hour"` is not part of the grammar.

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

## Recipe 8 — bounded set fan-out (design §2.a)

`read_many` reads a filtered set in ascending `id` order and binds it under
`as`. A following `for_each` runs its body once for each row. The hard
`FOR_EACH_MAX_ROWS` cap is **500**; there is no per-step override. Empty sets
are a no-op, and a cap violation or any body failure rolls back the whole
action, including writes before the loop.

The set has one addressable scalar, `$orders.count`: the non-null number of
rows bound for the loop (0–500). Use it in `returns`, `values`, or `set` when
the handler needs to report its fan-out size. Name that result
`rows_iterated`, not `rows_written`: the body may write a different number of
rows or roll back, and `.count` is not an accumulator.

To reach a child collection without nesting `for_each`, a later `where` may
use one property projection from the parent set as its direct `in` operand:
`{ "in": "$orders.id" }`. The parent must be a prior `read_many`, the property
must exist on its ObjectType, and its type must match the child filter column.
The projection is resolved to a bounded, parameterized `IN` list; when the
parent set is empty it compiles to `1=0`, never `IN ()`.

```json
{
  "steps": [
    { "op": "read_many", "object": "standing_order", "where": { "location_id": "$input.location_id" }, "as": "orders" },
    { "op": "for_each", "in": "$orders", "as": "order", "steps": [
      { "op": "insert", "object": "order_line", "values": { "order_id": "$order.id", "product_id": "$input.product_id", "quantity": "$input.quantity" } }
    ] }
  ],
  "returns": { "rows_iterated": "$orders.count" }
}
```

The same shape works for a child read or update:

```json
{
  "steps": [
    { "op": "read_many", "object": "order", "where": { "delivery_date": "$input.delivery_date" }, "as": "orders" },
    { "op": "read_many", "object": "order_line", "where": { "order_id": { "in": "$orders.id" } }, "as": "lines" }
  ],
  "returns": { "line_count": "$lines.count" }
}
```

For a bulk update, use the same second-step `where` with `op: "update"` and
provide its `set` map. A set binding is not a row: `$orders.id` in equality,
values, set, predicates, returns, or `for_each.in`, and a bare `$orders`
outside `for_each.in`, are declaration errors. The loop alias and body
bindings are scoped to the iteration. The alias cannot shadow an existing
binding or the reserved heads `input`, `principal`, `now`, and `actor`.

## Recipe 9 — effective dating and `standing_order.set` (design §2.b)

Use range operators for the lower bound and a flat `any_of` for an open-ended
or future upper bound. `absent` matches a `NULL` end date; `gt` keeps a record
whose end is after the instant being queried. For a latest-row lookup, add a
single-key `order_by` and `limit: 1`:

```json
{
  "steps": [
    {
      "op": "read_many",
      "object": "price",
      "where": {
        "valid_from": { "lte": "$input.as_of" },
        "valid_to": { "any_of": [ { "absent": true }, { "gt": "$input.as_of" } ] }
      },
      "order_by": { "property": "valid_from", "direction": "desc" },
      "limit": 1,
      "as": "prices"
    }
  ]
}
```

The two `where` fields are ANDed, while the `valid_to.any_of` members are ORed.
Keep the disjunction flat; nested `any_of` members are not part of the handler
grammar.

This is a complete `standing_order.set` handler. It treats periods as
half-open `[valid_from, valid_to)` intervals, so a new period may start exactly
when the previous one ends. The overlap read deliberately checks the EXISTING
row's `valid_to` with `any_of: [absent, gt]`: both a closed period that extends
past the requested start and an open-ended (`NULL`) period are clashes.

```json
{
  "steps": [
    {
      "op": "read_optional",
      "object": "standing_order",
      "where": {
        "location_id": "$input.location_id",
        "customer_id": "$input.customer_id",
        "product_id": "$input.product_id",
        "valid_from": { "lte": "$input.valid_from" },
        "valid_to": { "any_of": [ { "absent": true }, { "gt": "$input.valid_from" } ] }
      },
      "order_by": { "property": "valid_from", "direction": "desc" },
      "limit": 1,
      "as": "overlap"
    },
    {
      "op": "assert",
      "when": { "ref": "$overlap.id", "absent": true },
      "else": {
        "code": "VALIDATION",
        "field": "valid_from",
        "message": "standing order period overlaps an existing period"
      }
    },
    {
      "op": "insert",
      "object": "standing_order",
      "values": {
        "location_id": "$input.location_id",
        "customer_id": "$input.customer_id",
        "product_id": "$input.product_id",
        "mode": "$input.mode",
        "quantity": "$input.quantity",
        "valid_from": "$input.valid_from",
        "valid_to": "$input.valid_to"
      },
      "as": "created"
    },
    {
      "op": "read_optional",
      "when": { "ref": "$created.valid_to", "absent": true },
      "object": "standing_order",
      "where": {
        "location_id": "$input.location_id",
        "customer_id": "$input.customer_id",
        "product_id": "$input.product_id",
        "valid_from": "$input.valid_from",
        "valid_to": { "absent": true }
      },
      "as": "open_created"
    }
  ],
  "returns": {
    "id": "$created.id",
    "open_id": "$open_created.id"
  }
}
```

Declare `valid_to` as `{ "type": "string", "optional": true,
"nullable": true }`. When the caller omits it, the declared-optional input
binding writes `NULL`; the guarded second read then uses `IS NULL` to verify the
new open-ended row. Do not write `valid_to: { "lt": null }` or use a bound
`< NULL` comparison — SQL NULL never satisfies that predicate.

The reporter's sequence is therefore: call `standing_order.set` with quantity
25 and `valid_from: "2026-09-15"` (omit `valid_to`), call generated
`standing_order.update` with that id and `valid_to: "2026-12-31"`, then call
`standing_order.set` with quantity 30 and `valid_from: "2026-11-01"`; the last
call is rejected. Repeating the two `set` calls on another key without ending
the first period proves the open-ended case is rejected too.

## Recipe 10 — containment + date_part (design §§2.c–2.d)

Use `contains` when a declared JSON property stores an array of scalar values.
The property must be `type: "json"`; a text property is rejected by the
`contains_requires_json` typecheck rule. The operand may be a literal, a scalar
ref, or a scalar-producing expression such as `date_part`:

```json
{
  "steps": [
    {
      "op": "read_many",
      "object": "standing_order",
      "where": {
        "location_id": "$input.location_id",
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

In a `where`, membership is compiled as an `EXISTS` over SQLite `json_each`;
NULL cells produce no match and malformed JSON is a `VALIDATION` failure. In a
predicate, rows expose JSON text and the evaluator parses it; an input value
that is already an array is accepted. Membership uses type-strict equality, so
`[1, 2]` does not contain the string `"1"`. The operand must resolve to
`string`, `number`, `boolean`, `enum`, or `timestamp`; JSON and null operands
are rejected by `contains_operand_scalar`.

Pax-style `weekdays` columns must therefore be declared `type: "json"` even
though the underlying SQLite column remains TEXT. Declaring them as `string`
is intentionally rejected rather than silently applying text containment.

## Auto-CRUD on a compound root — the generated verbs change

The patterns above are for **custom** handlers you author. Auto-CRUD also
generates default actions per ObjectType, and for a plain (non-compound) type
that set is exactly five: `<type>.create`, `<type>.read`, `<type>.update`,
`<type>.delete`, `<type>.list` (each suppressible via `ObjectType.defaults`). The
moment a type declares a `part_of` link, the generated set grows and three of the
verbs gain compound behaviour (docs/adr/0002 — verified against
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
- **A predicate operand that cannot resolve** → `predicate_operand_unresolved`.
- **A predicate operand whose type is incompatible with its ref** →
  `predicate_operand_type_mismatch`.
- **`contains` on a non-JSON property** — `contains_requires_json`; declare
  JSON-array properties as `type: "json"`.
- **A JSON or null `contains` operand** — `contains_operand_scalar`; the
  membership needle must resolve to one scalar type.
- **An ordered predicate over a non-orderable ref** → `predicate_not_orderable`.
- **A `date_part` left operand that is not `timestamp` or `string`** →
  `date_part_left_not_date`.
- **An unsupported `date_part` part** (`hour`, for example) — rejected at
  decode because the part enum is closed.
- **An empty handler** — `steps` must be a non-empty array
  (`"handler must declare at least one step"`).
- **`read_aggregate` with zero or multiple aggregators** —
  `"read_aggregate requires EXACTLY ONE aggregator"`.
- **A set binding used as a row** (`$orders.id` or bare `$orders` in a
  predicate) — `set_binding_not_addressable`; use a set property only as the
  direct operand of a child `where.in`, or use the bare set as `for_each.in`.
- **An invalid `for_each` scope** — `for_each_in_not_set` when `in` is not a
  bare `read_many` binding, or `for_each_alias_shadow` when its alias collides
  with a binding or reserved head.
- **An empty `read_many.where`** — `read_many_where_empty`; every set read must
  be scoped by at least one equality field.
- **Assuming a handler `update` is single-row** — its `where` is set-wide and
  rewrites every matching row; only zero matches become `NOT_FOUND`. The DSL
  deliberately has no `expect: one` guard in this slice, so use a sufficiently
  identifying `where` when a write must affect one row.
- **A nested body error** reports its full path (for example,
  `steps[6].steps[3].where.valid_to`) and a compound label such as
  `for_each $orders › update order_line`.

### Set-extension rule reference

The set-valued extensions use stable error names and message prefixes. Every
one is an error (decode or `typecheck_failed`), never an advisory:

- `for_each_nesting` — `steps.<j>.op: Expected read | … actual for_each`.
- `set_binding_not_addressable` — `set binding $sos is only usable as for_each.in`.
- `where_in_set_binding_required` — `where.in operand must be a property of a read_many set binding`.
- `for_each_in_not_set` — `for_each.in must be a bare ref to a read_many set binding`.
- `for_each_alias_shadow` — `for_each alias $name shadows`.
- `for_each_body_scope` — `unknown step ref: $name (no prior step bound this name)`
  when a loop-local binding is used after the loop; outer bindings remain in scope.
- `read_many_where_empty` — `read_many requires a non-empty where`.
- `read_order_by_not_orderable` — `read_order_by_not_orderable: order_by.property must be a number, string, or timestamp property`.
- `read_limit_invalid` — `read_limit_invalid: limit must be a positive integer` (and, for `read_many`, no greater than 500).
- `where_op_exactly_one` — `where_op_exactly_one: where operator must carry exactly one operator`.
- `where_op_type_mismatch` — `where_op_type_mismatch: operand resolves to`.
- `where_op_not_orderable` — `where_op_not_orderable: operator '<op>' requires a number, string, or timestamp property`.
- `where_absent_on_non_nullable` — `where_absent_on_non_nullable: absent is a dead branch for non-nullable property`.
- `where_any_of_flat` — `where any_of members cannot contain any_of`.
- `predicate_any_of_flat` — `predicate any_of members cannot contain any_of`.
- `predicate_operand_unresolved` — `predicate_operand_unresolved:`.
- `predicate_operand_type_mismatch` — `predicate_operand_type_mismatch: operand resolves to`.
- `predicate_not_orderable` — `predicate_not_orderable: operator '<op>' requires a number, string, or timestamp ref`.
- `contains_requires_json` — `contains_requires_json: contains requires a json (JSON-array) property`.
- `contains_operand_scalar` — `contains_operand_scalar: contains operand must resolve to a scalar`.
- `date_part_left_not_date` — `date_part_left_not_date: date_part.left must resolve to 'timestamp' or 'string'`.

## Security gotcha: writing scope columns on a `target`-declared action

If your ActionType declares a write `target` (docs/adr/0005 — a derived-target
`actor_attr` or by-reference `input_key` scoping for an external write, e.g.
`guest.set_standing_line`), the external write gate scope-checks the
*resolved target row*, not your handler's `insert`/`update` `values`/`set`.
Concretely: **never bind a scope column (`customer_id`, an org-scoping
column, etc.) from `$input.<field>`** in a step that writes it — a client can
supply any value there and the gate will not catch it, because a declared
`target` bypasses the payload-candidate check entirely. Always derive scope
columns from `$principal.*` / the server-resolved target row (e.g. read the
target first with a `read` step keyed by the target id, then reference
`$<that-step>.customer_id` in your write), never from raw `$input.*`.
(The full writeup lives in the platform's ADR 0005, external-write-gate
target scoping.)

## See also

- [actions-input-governance.md](actions-input-governance.md) — the declaration envelope.
- Base skill: `canon skill --name canonify resource handler-dsl-cookbook.md` and the error-code reference `resource error-codes.md`.
