# The handler DSL

A handler is an **ordered list of typed steps over ObjectTypes**, glued by a
reference grammar, with an optional `returns` block and `on_fail`
compensator. It is deliberately small and closed — type-checked at declare
time against the live registry, not a programming language: no `if`/`else`
steps (two steps with opposite `when`s is the idiom; a value branches with
`case`), no function calls, no SQL (SQL lives in derived properties and the
read-only `sql_read`). Every step runs in one transaction with the runtime's
guard, audit and outbox around it; any failure rolls the whole action back. The
envelope around the handler is [actions-input-governance.md](actions-input-governance.md).

## Refs

| Ref | Resolves to |
|---|---|
| `$input.<field>` | a field of the decoded action input |
| `$<step>.<field>` | a field of a row a prior step bound with `as` |
| `$<read_many_step>` | a row set — usable as `for_each.in`; `$set.count` is its size; `$set.<property>` only as a direct `where.in` operand |
| `$now` | the org clock (its `demo_now` pin, else real time), ISO-8601 UTC |
| `$today` | the org's calendar day, a `date` (`YYYY-MM-DD`) on its `org.update { timezone }` (UTC when unset) |
| `$principal.id` / `.org_id` / `.kind` | the caller |
| `$principal.name` | the caller's name at invoke time — a member's org name (else email), a key's `"<label> (agent)"`; a snapshot. To show who acted, store `$principal.id` in a [`principal: true`](object-property-types.md#property-kinds) column instead |
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
| `read_aggregate` | exactly one of `count: true` / `sum` / `avg` / `min` / `max` over a `where`, optional `group_by`, or `peak_day` (the busiest day of a date range); binds `$<as>.<op>` (and `$<as>.day`) — or `per_day` (every day of a range), which binds `$<as>` as a day series for `for_each` | required |
| `insert` | a row from `values`; `id`, `created_at`, `updated_at` stamped; optional `attach` (below) | optional |
| `update` | every row matching `where` gets `set`; `updated_at` stamped; zero matches is `NOT_FOUND` unless `optional: true`; optional `attach` when it writes one row | — |
| `delete` | rows matching `where`, honouring links' `on_delete`; zero matches is a no-op | — |
| `assert` | aborts with `else: { code: VALIDATION \| FORBIDDEN \| INVALID_STATE, message, field?, on_item_fail? }` unless `when` holds; `reveals: true` lets it decide over columns the invoker cannot read (access-policies). The message may show values: `"{$r.name} cannot leave: {$r.lines_open} lines not checked"` — each `{$<binding>.<column>}` as the invoker reads it, `(withheld)` where they may not. A value no binding holds — one the step computes, such as the rest between two duties — is named in `else.values` and shown as `{$<name>}`: `"message": "Only {$rest} rest since the last duty", "values": { "rest": { "op": "date_diff", "left": "$input.report_at", "right": "$last.released_at", "unit": "m" } }` reads "Only 8 h 30 min rest since the last duty" (a name that is a binding or a reserved head is `text_value_shadows`; a `preCheck` guard takes none, `precheck_text_values`). Each value is written by its kind as a column is: a ref as its column, an input (`{$input.amount}`, or a field of an input line in a `for_each` over it) as its input's type — money in its currency, a date as a day, a set as its labels, never the submitted minor units or ISO text — a `date_diff` in minutes as a duration, a sum or difference as its first operand, an amount times or over a figure as the amount; a number with its property's `unit` and `scale.decimals` ("15.3 h"). A placeholder carries its unit, so the text never writes it again: `"{$s.hours} on this shift"`, not `"{$s.hours} hours on this shift"` (that reads "15.3 h hours"; `actions.declare` and `catalog validate` advise `doubled_unit`). A count takes its noun's form from the org language: `{$ev.uncovered|need is|needs are}` reads "1 need is" / "3 needs are" — never write "need(s)". A day is "9 Dec"; `{$x.day:weekday}` writes it as the page writes a near date, "Wed 9 Dec" (`:weekday` on a value that is no day is `guard_text_form_not_a_day`, another form `guard_text_form_unknown`). `else.severity: "warn"` (no `code`) lets the call run and answers the message on the envelope's `warnings` (top-level steps only; a warning over a value the invoker cannot read is not given); `actions.preview` of the same input answers the same `warnings` without saving, so a Spring UI dialog or Form page shows each text before Save (over 40 h this week, under 11 h rest). `preCheck: true` on a top-level refuse assert whose condition and placeholders read only the subject row (a `read` of the subject by `{ "id": "$input.<subject>" }`, compared with literals or relative times): a collection read (`objects.collection` with `verbs`) answers per row `__refused: { "<verb>": "<text>" }`, in the same read batch, so the verb shows disabled with the text before anyone clicks it; a guard over a column the reader may not read refuses them on every row alike unless it `reveals` | — |
| `transition` | names a row as a subject of the action's own transition marker; writes nothing itself | — |
| `call` | runs a declared fragment with inputs bound by `with` | optional |
| `for_each` | runs nested steps once per element of a `read_many` set or an `$input` array (`in`), kept by `where: { <field>: <value> }`; nests one level; `on_item_fail` reports each element (below) | the element |
| `pick_first` | binds the first row of an ordered set (`from`) that passes every `checks` step (below) | the row |
| `spread` | places an `amount` over an ordered `read_many` set (`over`), each row taking at most its `cap` column; binds the rows that took a share, with the declared `remainder` rule ([below](#spread--an-amount-over-rows-oldest-first)) | required |

`attach: { "<slot>": "$input.<file input>" }` binds the input's file — or
each of a `multiple: true` input's files — as an attachment of the row the
step wrote, under that slot, held to the type's `attachments` rules
([files](../../canonify-isomorphism/resources/runtime-contract.md)). A slot
named like a property of the type is `attach_slot_is_property`, a ref that is
no `file` input `attach_not_file_input`; an absent optional input attaches
nothing; an update that wrote several rows is refused (`VALIDATION`).

A step's `when` is a guard: the step runs only while it holds. An `assert`'s
`when` is the opposite — the rule that must hold, or the run aborts; to assert
only in some cases, guard a `call` of an asserting fragment (`"op": "call"`,
`"when": <the case>`). A fallback `update` that may find nothing to change
(`where: { settled_at: { absent: true } }`) declares `optional: true`.

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

At invoke, in one transaction: idempotency and governance checks, decode, the
guard (`lead.status` is `qualified`), the four steps (inserts stamp `id`,
`created_at`, `updated_at`), the runtime's `status = 'converted'`, the
`on_enter.converted` hooks to the outbox, the audit row. The envelope is
`{ data: { lead_id, customer_id, contact_id }, audit_event_id }`. A property the
written type lacks is refused at declare (`customer has no property 'x'`).

## Conditions — one grammar for every surface

One predicate over one op set serves a handler guard, every `showWhen` and
`emphasize[].when`, every filter (`source.filter`, `presets`, `inputFilters`,
`where`) and [to-cases](state-machines.md#to-cases). Each surface has one
spelling; another surface's is refused in one line that rewrites it.

| Op | Value | Holds when |
|---|---|---|
| `eq` / `neq` | a string, number, boolean or `null` | the value equals / does not equal it (`null` is blank; a blank equals no other value, so `neq` holds on it); on a list value (`$principal.groups`), an entry does / none does |
| `gt` `gte` `lt` `lte` | a number or string | both are numbers or both strings (ISO dates order as text); any other pairing never holds |
| `in` / `nin` | a non-empty list | the value is / is not one of them (a blank is one only if the list holds `null`) |
| `contains` | one scalar | a `json` / `set<enum>` list holds it, type-strict (`[1, 2]` lacks `"1"`) |
| `present` / `absent` | none (`true` in a filter) | the value is / is not set |

| Surface | Spelling | Example |
|---|---|---|
| handler `when` / `assert` / `case` | `{ ref, <op>: value }`, exactly one op | `{ "ref": "$order.status", "neq": "cancelled" }` |
| showWhen, `emphasize[].when`, a to-case `when` | a list of `{ field, op, value? }`, ANDed | `[{ "field": "status", "op": "neq", "value": "cancelled" }]` |
| filter | `{ <property>: <literal> \| { <op>: value } }`, keys ANDed; a literal is `eq`; ranges combine (`{ "gte": 1, "lt": 9 }`) | `{ "status": { "neq": "cancelled" }, "on_stop": false }` |

A literal is typed like its property — booleans `true` / `false`, money in
minor units, enums a declared value (`invalid_filter_value_type`,
`invalid_predicate_operand`). Groups, no `not` anywhere (write `neq`, `nin`, `absent`): a filter's `$or` / `$and` nest ≤ 3 deep; a handler guard nests ONE level — an `all_of` of leaves and `any_of` groups, an `any_of` of leaves only (`predicate_any_of_flat`). Write "A, or B and C" by distributing it: `{ "all_of": [{ "any_of": [A, B] }, { "any_of": [A, C] }] }`. Inside an action, every where (read, read_many, update, delete, transition) keeps SQL's rule: a blank fails `neq`/`nin` (add `absent` to include it); views, filters, presets and conditions treat a blank as not equal.

A condition list's value may be a relative time, resolved on the org clock as
a filter resolves it: `$now`, `$now±<duration>`, `$today`, `$today±<n>d`, or
a token (`:today`, `:this_week`, …) — `[{ "field": "starts_at", "op": "lte",
"value": "$now+2h" }]` offers check-in from two hours before the start. Use
`eq` / `neq` / an order (a day or token: `eq` is "on it", `lt` "before it").
Any other `$`-value is refused (`unresolved_condition_operand`) unless the
surface reads it (`$input.` / `$subject.` / `$reads.` in a form condition).

In a handler the operand may be a literal, a ref, an expression or a relative
time (`{ "ref": "$r.report_at", "lte": "$now+2h" }`), which resolves on the
org clock and compares as a time exactly as in a condition list and a
`preCheck` row read; a literal that is no valid ref (`{ "eq": "churned" }`)
compares verbatim. The compared side (`ref`) is a ref
or an expression too, under the same 8-level cap — `{ "ref": { "op": "+",
"left": "$s.booked", "right": "$input.seats" }, "lte": "$s.capacity" }` — so
`a + b <= c` is written as it reads; a literal or a list there is refused
(`condition_ref_not_a_value`). Each `any_of` member is a leaf
(`{ ref, <op> }`, never a group), and `any_of` members may carry different refs:
`{ "any_of": [{ "ref": "$input.guest_id", "absent": true }, { "ref": "$guest.id", "present": true }] }`.
`all_of` is guard-only; its members are leaves or `any_of` groups (else
`predicate_all_of_flat` / `predicate_any_of_flat`). Prefer it to chained
asserts, and always to count arithmetic (`eq: { "op": "-", "left": 2,
"right": "$locked.count" }` passes silently on one miscount).

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
to that field. Pick the code by cause: `VALIDATION` for bad input,
`FORBIDDEN` for who may act, `INVALID_STATE` for "not in this state" (the
transition guard's code — 409, audited `denied`). `CONFLICT` is the platform's
own (races, leases, terminal records — [error-codes.md](../../canonify/resources/error-codes.md#conflict-sub-codes)).

`contains` is a predicate operator: `when: { "ref": "$guest.tags", "contains": "vip" }`
never matches a NULL cell. A handler adds `has_any` / `has_all` on a `set<enum>`
— shares one of / holds all of the listed members; a set is an equality operand.
The operand is a list (`"has_all": ["mon", "$input.day"]`) or a ref to another
set, which compares set with set: "the route runs every day the customer
needs" is `{ "ref": "$r.run_days", "has_all": "$c.delivery_days" }` (an empty
or NULL operand set holds for `has_all`, never for `has_any`), in a `when`,
`assert`, `case` or `where` alike. A ref operand that is not a `set<enum>` is
`set_operator_requires_set_enum`; any other operand is `set_operand`.

### `where`

Each `where` key is a filter entry or one flat `any_of` of entries. A ref `in` /
`nin` operand is a prior `read_many` set's property or a `set<enum>` input (an
empty `in` is `1=0`; an empty `nin` is every row on a read, none on a write).
A filter is capped (1000 members per array, 10000 bound values, limited nesting): `where_too_large` / `where_too_deep`; `$as_of` on temporal types.

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

`iso_weekday` is a number, so this `weekdays` is `json` of ISO weekdays (`[1, 3]`, Monday = 1); weekdays people pick and read are a `set<enum>` of the `mon`…`sun` tokens the `weekday` part returns ([bookings](../../canonify-app-authoring/resources/patterns.md#bookings--free-slots-a-weekly-series-today)).

### Nullable bindings and narrowing

A `when`-guarded step binds its `as` to `null` when skipped, `read_optional` on
no row, `pick_first` when no row passes. A strict write position — `values`,
`set`, a bare `where` equality — takes such a binding only into a nullable
property (else `nullable_binding_into_non_nullable` names the alias, its guard
and the target); predicates stay null-tolerant. Narrowing is flow-sensitive:
`present: true` (or `neq: null`) on a binding narrows it for that step's maps,
on a `for_each` for the whole body, in an `assert` for every following step; an
`all_of` narrows what its top-level members prove, a leaf inside `any_of`
nothing. Only a guard on the binding itself narrows it — `$input.sample_id`
present proves nothing about `$sample`; write `{ "ref": "$sample.id",
"present": true }`. In `read_optional`/`read_many`/`read_aggregate` filters a
null binding becomes `IS NULL` (null, an empty set or zero).

Find-or-create:

```json
{ "steps": [
  { "op": "read_optional", "object": "customer", "where": { "email": "$input.email" }, "as": "existing" },
  { "op": "insert", "object": "customer", "values": { "name": "$input.name", "email": "$input.email" }, "as": "created", "when": { "ref": "$existing.id", "absent": true } } ] }
```

## Value expressions

Any value — a `values`/`set` field, a `where` or predicate operand, a `with`
binding — may be an expression node:

| Node | Shape |
|---|---|
| list | `["mon", "tue"]` — flat members, into a `set<enum>` / `json` property; never a comparison operand (`list_not_a_comparison_operand`: use `has_any` / `in`) |
| arithmetic | `{ "op": "+" \| "-" \| "*" \| "/" \| "%", "left", "right" }` (numeric operands; `money` counts; `%` = remainder) |
| text | `{ "op": "concat", "left", "right" }` — text, numbers or dates joined; null joins as `""` |
| rounding | `{ "op": "round" \| "floor" \| "ceil", "left", "decimals"? }` |
| date arithmetic | `{ "op": "date_add" \| "date_sub", "left", "right": <duration>, "tz"?, "business_days"?, "calendar"? }` |
| date part | `{ "op": "date_part", "part": "date" \| "iso_weekday" \| "year" \| "month" \| "day" \| "hour" \| "minute" \| "weekday" \| "time", "left", "tz"? }` — in the org zone; `date` is the `YYYY-MM-DD` day (as `$today` / `:today` name today's), `weekday` is `mon`…`sun`, `time` is `HH:MM` (compare it with a `time` property) |
| date difference | `{ "op": "date_diff", "left", "right", "unit": "m" \| "h" \| "d" }` — `left` − `right` (two timestamps, two dates, or two `time`s of one day) in whole elapsed units, truncated toward zero; a number (in minutes it is written as a duration, "8 h 30 min") |
| instant of a date and a time | `{ "op": "org_instant", "left": <date>, "right": <time> }` — the timestamp the org's clock reads that day at that time, DST included (the handler twin of the derived-SQL `org_instant(date, time)`); never `concat(date, "T", time)`, which names no zone |
| case | `{ "case": [{ "when": <predicate>, "then": <value> }, …], "else": <value> }` — the first arm whose `when` holds |

Operands nest: any `left`/`right` may itself be an expression, nested at most 8
levels (`value_expr_too_deep` beyond: compute a part in a fragment `call` and
ref its `returns`); `has_any`/`has_all` members stay flat (or the operand is one ref to a set).
A `case` guard is the step-`when` predicate grammar ([below](#conditions--when-assert-predicates));
every `then` and the `else` resolve to one type (`case_branch_type_mismatch`).
A `date_add`/`date_sub` `right` is a duration: a literal (`"3d"`), a counted
`{ "n": <value>, "unit": "mo" }` whose `n` must be a number
(`duration_count_not_number`) and, at run time, a whole number (a division is
`duration_count_fractional`: wrap it in `round`); a computed negative count moves
the other way (`date_add` of `-60` minutes is an hour earlier); or a `case` of durations:

```json
{ "op": "date_add", "left": "$now", "right": { "case": [
  { "when": { "ref": "$input.priority", "eq": "urgent" }, "then": "1d" } ],
  "else": { "n": "$input.lead_days", "unit": "d" } } }
```

```json
{ "op": "round", "decimals": 2, "left": { "op": "/", "right": 100,
  "left": { "op": "*", "left": "$price.list_price",
    "right": { "op": "-", "left": 100, "right": "$input.discount_pct" } } } }
```

`73.75 × (100 − 10) / 100 = 66.375` freezes as `66.38` (depth 4): `round` shifts
by `10^decimals` (0–6, default 0) and is half-up away from zero (`round(-2.5)` =
`-3`, `round(1.005, 2)` = `1.01`); `floor`/`ceil` are the math functions. Money
stays in minor units; a money column refuses a fractional result — round first.

**Dates.** Durations are `30s` · `15m` · `1h` · `1d` · `1w` · `1mo` · `1y` (`m` =
minutes, `mo` = months; `Nw` = 7N days; a month or year step clamps to the
month's end: 2026-01-31 + `1mo` = 2026-02-28) — one grammar with filters'
`$now±…` and schedules. Without `tz`, arithmetic is UTC milliseconds and every
`date_part` reads the org zone (UTC if it has none; `"tz": "UTC"` pins UTC). On arithmetic,
`tz: "org"` or an IANA zone (checked at declare) keeps `d`/`w` on the local wall-clock time (`s`/`m`/`h` stay
elapsed); a spring-forward gap moves forward (02:30 → 03:30), an autumn-back
overlap takes the earlier instant. `business_days: true` (only `d`/`w`) skips
local Saturday/Sunday and the dates of a `calendar` ObjectType (`Nw` = 5N
business days), which must declare mapped `date` and `org_id` properties (rows
match the caller's org; no global calendar). `date_part` of a `YYYY-MM-DD` is that
date (`date_part_left_not_date` for a non-date; `date_part_no_time_of_day` for its `hour`).

`date_diff` is elapsed time, so a DST switch between the two counts the hour it
adds or takes: in Vienna, released 22:30 CEST on Saturday 24 October and
reporting 06:30 CET on Sunday is 8 h on the wall clock and 9 h of rest. Two
dates differ by calendar days (`"unit": "d"`); a date against a timestamp is
`date_diff_mixed` (compare the timestamp's `date_part` `date`).
Two `time`s (`HH:MM`) are the wall-clock minutes between them on one day — negative when `left` is the earlier, `date_diff_time_unit` in `"d"`; a time against a date or a timestamp is `date_diff_mixed` (make the instant with `org_instant`). A same-day shift is one `date` and two `time` inputs: `"paid_minutes": { "op": "date_diff", "left": "$input.end_time", "right": "$input.start_time", "unit": "m" }` (assert it `gt` 0) and `"starts_at": { "op": "org_instant", "left": "$input.date", "right": "$input.start_time" }` — 09:00–17:30 is 510 minutes ("8 h 30 min" under `unit: "min"`), starting 08:00Z in Lisbon in July; past midnight, a `case` adds 1440 when the end is the earlier time.
A null operand
refuses (`VALIDATION`), so an `any_of` puts the `absent` arm first. The rest
between two duties, refused under 9 h and warned under 11 h, with the rest it
found in the text (shown as "8 h 30 min"; a `number` property with
`unit: "min"` is written the same way):

```json
{ "steps": [
  { "op": "read_optional", "object": "duty", "as": "last",
    "where": { "driver_id": "$input.driver_id", "released_at": { "lte": "$input.report_at" } },
    "order_by": { "property": "released_at", "direction": "desc" }, "limit": 1 },
  { "op": "assert",
    "when": { "any_of": [
      { "ref": "$last.id", "absent": true },
      { "ref": { "op": "date_diff", "left": "$input.report_at", "right": "$last.released_at", "unit": "m" }, "gte": 540 } ] },
    "else": { "code": "VALIDATION", "field": "report_at",
      "message": "Only {$rest} rest since the last duty: 9 h is the least.",
      "values": { "rest": { "op": "date_diff", "left": "$input.report_at", "right": "$last.released_at", "unit": "m" } } } },
  { "op": "assert",
    "when": { "any_of": [
      { "ref": "$last.id", "absent": true },
      { "ref": { "op": "date_diff", "left": "$input.report_at", "right": "$last.released_at", "unit": "m" }, "gte": 660 } ] },
    "else": { "severity": "warn", "message": "Only {$rest} rest since the last duty (under 11 h).",
      "values": { "rest": { "op": "date_diff", "left": "$input.report_at", "right": "$last.released_at", "unit": "m" } } } }
] }
```

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

Over no rows a `count` or `sum` is `0`; an `avg`, `min` or `max` has no value
(`null`), and the type-check says so: writing it into a property that is not
nullable is `nullable_binding_into_non_nullable`, and computing with it
(`+ - * / %`, `round`) is `nullable_operand`, until a `when` on the step or a
`case` arm proves it present. Default it with a `case`:
`{ "case": [{ "when": { "ref": "$last.max", "present": true }, "then": "$last.max" }], "else": 0 }`.

```json
{
  "steps": [
    { "op": "read_aggregate", "object": "order", "where": { "customer_id": "$input.customer_id" }, "sum": "amount_cents", "as": "revenue" },
    { "op": "update", "object": "customer", "where": { "id": "$input.customer_id" }, "set": { "lifetime_cents": "$revenue.sum" } }
  ]
}
```

**Each day of a range — `peak_day`, `per_day`.** A rule "on no day may the
holds exceed what is owned" needs each day's total, not the range's. On a
`count` or `sum`, `peak_day: { span: [<start date>, <end date>], from, to }`
counts a row on every day of `from`…`to` (at most 400) its two date properties
cover (a blank end covers its start day) and binds the busiest day's value as
`$<as>.<op>` and that day as `$<as>.day` (the earliest of equal days; `0` and
`null` when the range is empty), so the refusal names the day that does not
fit. When the day's limit is itself a series (owned plus what is hired in that
day), `per_day` (same keys) binds `$<as>` as the day series — `{ day, <op> }`
for every day in order, `0` where nothing holds — for a `for_each`, and a
nested `for_each` with `where: { "day": "$d.day" }` joins another series:

```json
{ "steps": [
  { "op": "read", "object": "kit_item", "where": { "id": "$input.kit_item_id" }, "as": "it" },
  { "op": "read_aggregate", "object": "kit_reservation", "where": { "kit_item_id": "$input.kit_item_id" }, "sum": "qty",
    "per_day": { "span": ["out_on", "back_on"], "from": "$input.out_on", "to": "$input.back_on" }, "as": "held" },
  { "op": "read_aggregate", "object": "kit_hire", "where": { "kit_item_id": "$input.kit_item_id" }, "sum": "qty",
    "per_day": { "span": ["from_on", "to_on"], "from": "$input.out_on", "to": "$input.back_on" }, "as": "hired" },
  { "op": "for_each", "in": "$held", "as": "d", "steps": [
    { "op": "for_each", "in": "$hired", "where": { "day": "$d.day" }, "as": "h", "steps": [
      { "op": "assert",
        "when": { "ref": { "op": "+", "left": "$d.sum", "right": "$input.qty" }, "lte": { "op": "+", "left": "$it.owned_qty", "right": "$h.sum" } },
        "else": { "code": "VALIDATION", "field": "qty", "values": { "q": "$input.qty" },
          "message": "Over-booked on {$d.day:weekday}: {$it.name} — {$it.owned_qty} owned + {$h.sum} hired in, {$d.sum} already held, {$q} asked." } } ] } ] } ] }
```

A day is written "24 Nov" (`:weekday`: "Tue 24 Nov"). Neither takes `group_by`
or `avg`/`min`/`max` (`peak_day_op`, `per_day_op`); `span` is two `date`
properties; a series is read only as `for_each.in` (or `.count`); both on one
read is `per_day_with_peak_day`. The page shows the same days with an Aggregate
over its range State (canonify-springml, recipes: 8. Availability).

Minting a sequential human code (`J-1042`) with `max` + 1 and a derived
property is in [canonify-app-authoring › patterns.md](../../canonify-app-authoring/resources/patterns.md).

## Sets — `read_many` and `for_each`

`read_many` binds at most **500** rows; more fail the action with `VALIDATION`
`details: { code: "row_cap_exceeded", step, object, as, cap: 500 }` (never a
silent truncation); `limit` (1–500, or a required `$input` number declared `min` ≥ 1, `max` ≤ 500) takes a subset. Its `where` must not be
empty (`read_many_where_empty`). `order_by` is one `{ property, direction }`,
or a list of up to three applied in turn, over number, money, string, timestamp
or date properties (derived too); on `read_optional` it picks the row. `$set.count` is the rows bound. A child set
is one `where.in` read; a loop over it nests in the parent's, `where`-kept:

```json
{
  "steps": [
    { "op": "read_many", "object": "order", "where": { "delivery_date": "$input.delivery_date" }, "as": "orders" },
    { "op": "read_many", "object": "order_line", "where": { "order_id": { "in": "$orders.id" } }, "as": "lines" },
    { "op": "for_each", "in": "$orders", "as": "order", "steps": [
      { "op": "insert", "object": "pick_list", "values": { "order_id": "$order.id" }, "as": "pick" },
      { "op": "for_each", "in": "$lines", "as": "line", "where": { "order_id": "$order.id" }, "steps": [
        { "op": "insert", "object": "pick_line", "values": { "pick_id": "$pick.id", "qty": "$line.qty" } } ] }
    ] } ],
  "returns": { "made": "$orders.count" }
}
```

Both levels' rows are ≤ 500 together, counted before any write; a nested
`where` reads `$input`, the outer row or earlier steps (`for_each_where_invalid`);
a third level is `for_each_nesting_too_deep`. An empty set is a no-op. An alias
may not shadow a binding or `input`, `principal`, `now`, `actor`
(`for_each_alias_shadow`); loop-local bindings end with the loop. A set used as a row (`$orders.id` outside a
`where.in`, bare `$orders` outside `for_each.in`) is `set_binding_not_addressable`.
A header + lines order is one action: `insert` the header `as: "order"`, then
`for_each` `in: "$input.lines"` (an `array` input) `as: "line"` inserting
`{ "order_id": "$order.id", "qty": "$line.qty" }`; `$line.<field>` is typed from
`items` (an omitted optional one is `null`); else `for_each_in_not_rows`.

**Set transitions.** A `transition` step inside a `for_each` names each row the
action's marker moves; the runtime guards, writes and fires hooks per row after
the handler, atomically ([state-machines.md](state-machines.md)). Insert +
`transition` in one action is in [patterns.md](../../canonify-app-authoring/resources/patterns.md).

**Per-item outcomes.** A `for_each` with `on_item_fail: "skip" | "refuse"` runs each element in a
savepoint: one whose assert, guarded `call` or database constraint fails rolls
back alone — `skip` commits the rest, `refuse` refuses the run (as without it) —
and an assert's own `else.on_item_fail` overrides the loop (a closure `skip`s in
a `refuse` series). The result carries `outcomes: [{ step, index, pick?, input?, id?,
item, status: "ok" | "skipped" | "refused", reason }]` (`item`: the loop's `item`
map, else the `$input` element or the row's `{ id }`; a row the caller cannot read: `id: null`, `item: { hidden: true }`;
a declared `item` is not column-masked, so name only fields every caller may see), and a refusal carries those
reported before it in `error.details.outcomes`; `actions.preview` lists
every element and writes nothing, and the form shows the list (under a refusal) with Drop or Move (its date fields) on a
refused `$input` element. `pick_first: { from, as, checks, item? }` binds `$<as>`
to the first row of an ordered set (a `read_many` with `order_by`, or an `$input`
array) passing every check (asserts, `call`s) — `null` if none — and reports each
earlier row `skipped` (`pick: true`: "Passed over") with its failing check's message (the picked one `ok`). A
waitlist offer names each passed-over patient (fragment `offer_fits` in [bookings](../../canonify-app-authoring/resources/patterns.md#bookings--free-slots-a-weekly-series-today)):

```json
{ "op": "pick_first", "from": "$waiting", "as": "next", "item": { "patient_id": "$next.patient_id" }, "checks": [
  { "op": "call", "fragment": "offer_fits", "with": { "entry_id": "$next.id", "entry_practitioner_id": "$next.practitioner_id",
    "slot_id": "$slot.id", "slot_practitioner_id": "$slot.practitioner_id" } } ] }
```

## `spread` — an amount over rows, oldest first

A payment that pays the oldest open charge first, a delivery split over
orders, a budget over lines: one amount placed over an ordered set, each row
taking at most what it can still hold. `spread` does the accumulation; the
writes stay in a `for_each` over its result.

```json
{
  "steps": [
    { "op": "read_many", "object": "charge", "as": "open",
      "where": { "trip_id": "$input.trip_id", "is_open": true },
      "order_by": { "property": "due_on", "direction": "asc" } },
    { "op": "spread", "amount": "$input.amount_cents", "over": "$open",
      "cap": "open_cents", "remainder": "refuse", "as": "paid",
      "else": { "code": "VALIDATION", "message": "More than is open on the trip: {$paid.total} open.", "field": "amount_cents" } },
    { "op": "for_each", "in": "$paid", "as": "c", "steps": [
      { "op": "insert", "object": "payment_allocation",
        "values": { "charge_id": "$c.id", "amount_cents": "$c.share" } },
      { "op": "update", "object": "charge", "where": { "id": "$c.id" },
        "set": { "paid_cents": { "op": "+", "left": "$c.paid_cents", "right": "$c.share" } } } ] }
  ],
  "returns": { "charges_paid": "$paid.count", "applied_cents": "$paid.total" }
}
```

- `over` is a bare `read_many` binding with an `order_by` — the rows fill in
  that order (`spread_over_not_rows`, `spread_over_unordered`); `cap` is a
  `number` or `money` property of its type, what each row can still take
  (`spread_cap_not_amount`; a blank or negative cap takes nothing); `amount`
  is a number or money value (`spread_amount_not_number`), at run time zero or
  more.
- `$<as>` is the set of rows that took a share, in order, each with its
  columns and `.share` (typed like the cap; a type with a `share` property is
  `spread_share_shadows`). `$<as>.count` is how many, `$<as>.total` the amount
  placed, `$<as>.remainder` what the rows could not hold. A row that took
  nothing is not in the set.
- `remainder` is required — the rule for an amount larger than the rows hold:
  `"refuse"` fails the run (`VALIDATION`, `details.code: spread_remainder`,
  and `failed_precondition.field` the amount's input) with `else` — `{$ref}`s
  over earlier bindings or `$<as>.total` / `.remainder`, each written by its
  kind — or "€ 1,00 would be left unallocated: the 3 charge rows hold
  € 6,00."; `"keep"` binds it as `$<as>.remainder` for the handler to place (a
  credit row under a `when`), and takes no `else` (`spread_else_without_refuse`).
- The hand-unrolled loop is refused at declare (`unrolled_loop`): a
  `read_optional` of the first row by a `where` and an `order_by`, repeated
  after a step that wrote that type, pays as many rows as it was copied for and
  leaves every later row untouched with no error. Spread over the
  `read_many`, or `for_each` over it.

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
- Re-declaring needs `expectedHash` (from `catalog.export`) or `force`. A
  change that breaks a caller (`fragment_caller_broken`, in a bundle too) or a
  prune of a fragment a surviving action calls (`unknown_fragment` /
  `fragment_in_use`) is refused before any write.

## `on_fail`

Names an action to enqueue on the outbox when an `assert` or the transition
guard rejects. Its `inputs` may use `$input`, `$principal`, `$now` and `$today`, never a
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
| `for_each_nesting_too_deep` · `for_each_where_invalid` · `for_each_in_not_set` · `for_each_in_not_rows` · `for_each_alias_shadow` · `for_each_body_scope` · `on_item_fail_outside_loop` · `on_item_fail_nested` · `returns_outcomes_reserved` · `pick_first_from_not_rows` · `pick_first_from_unordered` · `pick_first_alias_shadow` | loop and per-item outcome misuse |
| `set_binding_not_addressable` · `where_in_set_binding_required` · `read_many_where_empty` · `read_limit_invalid` · `read_limit_input_unbounded` | set misuse; `limit` must be a positive integer (≤ 500 on `read_many`), or a required `$input` number with `min` ≥ 1 and `max` ≤ 500 |
| `read_order_by_not_orderable` · `where_op_not_orderable` · `predicate_not_orderable` | ordering a non-orderable property or ref |
| `where_op_combination` · `where_op_type_mismatch` · `where_absent_on_non_nullable` · `where_any_of_flat` | `where` operator misuse |
| `predicate_operand_unresolved` · `predicate_operand_type_mismatch` · `predicate_any_of_flat` · `predicate_all_of_flat` | predicate misuse |
| `contains_requires_json` · `contains_operand_scalar` · `set_operator_requires_set_enum` | membership misuse |
| `value_expr_too_deep` · non-numeric arithmetic · bad duration · `case_branch_type_mismatch` · `duration_count_not_number` | expression misuse |
| `file_list_into_property` · `attach_slot_is_property` · `attach_not_file_input` | a `multiple` file input stored into a `file` property; `attach` misuse |
| `reveals_text_interpolates` · `reveals_not_a_guard` | a revealing assert's message names a value; `reveals` on an optional `update` |
| `assert_code_missing` · `assert_warning_code` · `assert_warning_placement` · `precheck_placement` · `assert_text_unknown_ref` · `invalid_precheck` | a refusing assert with no `code`; a warning with one, or inside a loop / pick_first / fragment; a `preCheck` on a warning or nested step; a `{$ref}` in the message over no binding; a `preCheck` the subject row alone cannot decide (no subject, another binding, a non-literal operand, `has_any` / `has_all`) |
| `date_part_left_not_date` · `date_part_no_time_of_day` · unknown `part` (decode) | date part misuse |
| `date_diff_operand_not_date` · `date_diff_mixed` · `date_diff_time_unit` · unknown `unit` (decode) | a `date_diff` side that is no date, timestamp or time; a date against a timestamp, or a time against either; two times counted in days |
| `org_instant_operand` | an `org_instant` whose `left` is no date or whose `right` is no time |
| `spread_over_not_rows` · `spread_over_unordered` · `spread_cap_not_amount` · `spread_amount_not_number` · `spread_alias_shadow` · `spread_share_shadows` · `spread_else_without_refuse` | `spread` misuse |
| `unrolled_loop` | a first-row `read_optional` repeated after a write to its type — a loop unrolled by hand; use `spread` or `for_each` |
| `text_value_shadows` · `precheck_text_values` | an `else.values` name that is a binding or reserved head; `else.values` on a `preCheck` guard |
| `nullable_binding_into_non_nullable` | a maybe-null binding into a required write |
| `enum_from_free_text` · `enum_value_unknown` | free text (a `string` input or column, `$principal`, a `concat`) written into an `enum` property with nothing proving it one of the values — declare the input `{ "type": "enum", "values": [...] }`, prove it with an `assert` or the step's `when` (`{ "ref": "$input.reason", "in": [...] }`), or map it with a `case` whose arms write the values; a literal or another enum's value outside the property's values. A value the check cannot bound (`json`) is refused when written: `VALIDATION`, `field_errors` naming the input |
| `nullable_operand` | arithmetic on an `avg` / `min` / `max` aggregate no guard proves present (over no rows it has no value) |
| `unknown_fragment` · `call_input_missing` · `call_input_unknown` · `call_returns_no_field` | fragment misuse |
</content>
