# Pricing, end to end

Obligation 4 as one continuous build: tables, refinements, the resolution
action, the settlement action that snapshots, and the sealing FSM. Every
declaration here is something you can invoke through `canon`, REST, or MCP.

The domain: products have a **list price**; some customers have a **negotiated
override**, either an absolute amount or a percentage off list; orders settle
into lines that must reproduce forever; and at month end somebody hands
invoicing a **sealed report**.

Two conventions apply throughout, and both belong in every ObjectType
description you write here:

- **Half-open intervals.** `valid_from` inclusive, `valid_to` exclusive. In
  effect at `:at` means `valid_from <= :at AND :at < valid_to`.
- **Open-end sentinel.** `valid_to = '9999-12-31T00:00:00.000Z'` means still in
  force. Never `NULL`.

---

## Step 1 — The tables

```sql
CREATE TABLE product (
  id          TEXT PRIMARY KEY,
  sku         TEXT NOT NULL,
  name        TEXT NOT NULL,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
)
```

```sql
-- The list price. Effective-dated, append-only.
CREATE TABLE price_list_item (
  id           TEXT PRIMARY KEY,
  product_id   TEXT NOT NULL,
  amount_minor INTEGER NOT NULL,   -- øre. Never REAL. (C10-float-money)
  currency     TEXT NOT NULL,
  valid_from   TEXT NOT NULL,
  valid_to     TEXT NOT NULL,
  note         TEXT,               -- why this price changed. Future-you asks.
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)
```

```sql
-- The negotiated override. ONE of amount_minor / percent_off is set.
CREATE TABLE customer_price (
  id           TEXT PRIMARY KEY,
  customer_id  TEXT NOT NULL,
  product_id   TEXT NOT NULL,
  amount_minor INTEGER,            -- absolute agreed price, or NULL
  percent_off  INTEGER,            -- basis points off list, or NULL
  currency     TEXT NOT NULL,
  valid_from   TEXT NOT NULL,
  valid_to     TEXT NOT NULL,
  agreement_ref TEXT,              -- the contract clause this came from
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)
```

> **`percent_off` is basis points, an INTEGER.** 12% is `1200`. The reason is
> the same as for money: a percentage stored as `0.12` is a float in the middle
> of a money calculation, and floats do not belong there. Basis points keep the
> entire computation in integers until the single rounding step.

> **A row with `amount_minor: 0` is a deliberately free line.** It is not the
> same as no `customer_price` row at all, which means "no agreement — fall
> through to list". `amount_minor` is nullable here *only* because it is the
> exclusive alternative to `percent_off`; it is never NULL as a way of saying
> "no price". See Obligation 4.3.

```sql
CREATE TABLE customer_order (
  id           TEXT PRIMARY KEY,
  customer_id  TEXT NOT NULL,
  placed_at    TEXT NOT NULL,      -- THE resolution date. Not $now at billing time.
  status       TEXT NOT NULL DEFAULT 'draft',
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)
```

```sql
CREATE TABLE order_line (
  id                 TEXT PRIMARY KEY,
  order_id           TEXT NOT NULL,
  product_id         TEXT NOT NULL,
  quantity           INTEGER NOT NULL,

  -- THE SNAPSHOT — what we actually charged, frozen at settlement.
  unit_amount_minor  INTEGER NOT NULL,
  line_amount_minor  INTEGER NOT NULL,
  currency           TEXT NOT NULL,

  -- THE BACK-REFERENCE — which agreement produced it.
  price_source_kind  TEXT NOT NULL,   -- 'customer_price' | 'price_list_item'
  price_source_id    TEXT NOT NULL,
  list_price_id      TEXT,            -- set when a percentage was applied
  percent_off        INTEGER,         -- the basis points applied, if any
  priced_at          TEXT NOT NULL,   -- the instant resolution ran against

  created_at         TEXT NOT NULL,
  updated_at         TEXT NOT NULL
)
```

Both halves are mandatory. The snapshot answers *what did we charge?*; the
back-reference answers *on the authority of what?*; and the dated rows, still
sitting untouched in `price_list_item` / `customer_price`, answer *what was the
agreed price on 3 March?* — a question the snapshot cannot answer, because it
only knows the one line it is on.

---

## Step 2 — Refine the ObjectTypes (anchor everything)

Every `*_id` column above gets an upward link. This is Obligation 2, and
skipping it here is what makes a billing model unnavigable later: you cannot
click from a line to the price row that justified it.

```yaml
action: catalog.refine_object_type
input:
  name: price_list_item
  description: >-
    List price for a product, effective-dated. APPEND-ONLY: never update
    amount_minor and never delete a row — close the current interval
    (valid_to := the successor's valid_from) and insert a new one.
    valid_from inclusive, valid_to exclusive, '9999-12-31T00:00:00.000Z' = open.
  links:
    product: { target: product, cardinality: one, via: product_id, fk_on: source }
  properties:
    amount_minor: { kind: mapped, column: amount_minor, type: number, currency: DKK }
    valid_from:   { kind: mapped, column: valid_from,   type: timestamp }
    valid_to:     { kind: mapped, column: valid_to,     type: timestamp }
```

```yaml
action: catalog.refine_object_type
input:
  name: customer_price
  description: >-
    Negotiated price or discount for one (customer, product), effective-dated
    and append-only. Exactly one of amount_minor / percent_off is set.
    percent_off is BASIS POINTS (1200 = 12%). amount_minor 0 means
    deliberately free; a missing row means no agreement.
  links:
    customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
    product:  { target: product,  cardinality: one, via: product_id,  fk_on: source }
  properties:
    amount_minor: { kind: mapped, column: amount_minor, type: number, currency: DKK }
```

```yaml
action: catalog.refine_object_type
input:
  name: order_line
  description: >-
    A settled order line. unit_amount_minor / line_amount_minor are a SNAPSHOT
    frozen at settlement and are never recomputed; price_source_id points at
    the effective-dated row that produced them. Rounding: half-up to whole øre,
    applied once per line; the order total is the integer sum of line amounts
    and is never rounded again.
  links:
    order:        { target: customer_order,  cardinality: one, via: order_id,   fk_on: source }
    product:      { target: product,         cardinality: one, via: product_id, fk_on: source }
    list_price:   { target: price_list_item, cardinality: one, via: list_price_id, fk_on: source }
  properties:
    unit_amount_minor: { kind: mapped, column: unit_amount_minor, type: number, currency: DKK }
    line_amount_minor: { kind: mapped, column: line_amount_minor, type: number, currency: DKK }
    price_source_kind:
      kind: mapped
      column: price_source_kind
      type: enum
      values: [customer_price, price_list_item]
```

Note where the rounding policy lives: **in the ObjectType description**, in
prose, saying where it is applied, to how many places, and in which direction.
That sentence is the thing an accountant asks for when a total is 3 øre off, and
it costs nothing to write today.

---

## Step 3 — Repricing (append-only, in one transaction)

```json
{
  "name": "price_list_item.reprice",
  "description": "Close the open list-price interval and open a new one. Never mutates an amount.",
  "object_type": "price_list_item",
  "input": {
    "product_id":   { "type": "string" },
    "amount_minor": { "type": "number" },
    "valid_from":   { "type": "string", "format": "iso8601" },
    "note":         { "type": "string", "optional": true }
  },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent"] },
  "idempotency": "per-key",
  "handler": {
    "steps": [
      { "op": "read", "object": "price_list_item",
        "where": { "product_id": "$input.product_id", "valid_to": "9999-12-31T00:00:00.000Z" },
        "as": "current" },
      { "op": "update", "object": "price_list_item",
        "where": { "id": "$current.id" },
        "set": { "valid_to": "$input.valid_from" } },
      { "op": "insert", "object": "price_list_item",
        "values": {
          "product_id": "$input.product_id",
          "amount_minor": "$input.amount_minor",
          "currency": "$current.currency",
          "valid_from": "$input.valid_from",
          "valid_to": "9999-12-31T00:00:00.000Z",
          "note": "$input.note"
        },
        "as": "next" }
    ],
    "returns": { "closed_id": "$current.id", "opened_id": "$next.id" }
  }
}
```

Three things to notice:

- The `update` sets `valid_to` and nothing else. **`amount_minor` is never
  written twice.** That is the append-only invariant, and it is what makes the
  history reconstructable.
- Close and insert are steps in **one** action, so they share the runtime's
  transaction. A close that commits without its insert leaves the product with
  no price in force.
- `valid_from` is an input, not `$now`. That is what makes a price schedulable:
  agree it today, effective 1 April, and nothing has to happen on 1 April.

The first price for a product is a plain `price_list_item.create` with
`valid_from` = the day it takes effect and `valid_to` = the sentinel. Only
subsequent changes go through `reprice`.

---

## Step 4 — Resolution

Resolution is a **two-case lookup**, in order. Do not build a rules engine for
two cases.

```
resolve(product_id, customer_id, at):

  1. customer_price where customer_id = :customer_id
                      and product_id  = :product_id
                      and valid_from <= :at and :at < valid_to
     found?  → if amount_minor is set:  unit = amount_minor
               if percent_off is set:   unit = apply(list(product_id, at), percent_off)
               (kind = 'customer_price', source_id = that row's id)

  2. otherwise list(product_id, at)
     (kind = 'price_list_item', source_id = that row's id)

  neither?  → refuse. VALIDATION, "no price in effect for <product> at <at>".
             Never default to 0, and never fall back to "today's price".
```

The `at` you pass is **`customer_order.placed_at`** — the instant the
transaction happened. Passing `$now` is the single most common way to
accidentally rewrite history: re-run last month's billing after a repricing and
every line silently changes.

Note that case 1 with `percent_off` reaches into case 2: a percentage agreement
is meaningless without the list price **on that same date**. This is the reason
§4.1's append-only rule is load-bearing rather than merely tidy — update a list
price in place and every historical percentage agreement recomputes against a
number that did not exist at the time.

`apply(list_amount, percent_off)` is where the rounding policy fires, exactly
once:

```
apply(list_amount_minor, percent_off_bp) =
    round_half_up( list_amount_minor * (10000 - percent_off_bp) / 10000 )
```

Everything on the right-hand side is an integer until the single `round_half_up`
divides. That is deliberate: no intermediate float, one defined rounding, one
reproducible answer.

---

## Step 5 — Settlement (the snapshot)

```json
{
  "name": "order_line.settle",
  "description": "Freeze the resolved unit price onto the line, with a reference to the row it came from.",
  "object_type": "order_line",
  "input": {
    "order_id":          { "type": "string" },
    "product_id":        { "type": "string" },
    "quantity":          { "type": "number" },
    "unit_amount_minor": { "type": "number" },
    "line_amount_minor": { "type": "number" },
    "currency":          { "type": "string" },
    "price_source_kind": { "type": "string", "enum": ["customer_price", "price_list_item"] },
    "price_source_id":   { "type": "string" },
    "list_price_id":     { "type": "string", "optional": true },
    "percent_off":       { "type": "number", "optional": true },
    "priced_at":         { "type": "string", "format": "iso8601" }
  },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent", "service_account"] },
  "idempotency": "per-key",
  "handler": {
    "steps": [
      { "op": "read", "object": "customer_order", "where": { "id": "$input.order_id" }, "as": "order" },
      { "op": "insert", "object": "order_line",
        "values": {
          "order_id": "$input.order_id",
          "product_id": "$input.product_id",
          "quantity": "$input.quantity",
          "unit_amount_minor": "$input.unit_amount_minor",
          "line_amount_minor": "$input.line_amount_minor",
          "currency": "$input.currency",
          "price_source_kind": "$input.price_source_kind",
          "price_source_id": "$input.price_source_id",
          "list_price_id": "$input.list_price_id",
          "percent_off": "$input.percent_off",
          "priced_at": "$input.priced_at"
        },
        "as": "line" }
    ],
    "returns": { "line_id": "$line.id" }
  }
}
```

After this, the line is a **fact**. Reprice the product tomorrow, renegotiate
the customer's discount next month, delete nothing — the line still says what it
charged, and `price_source_id` still points at the interval that justified it,
because that interval was closed rather than overwritten.

Two questions, two answers, and they do not compete:

| Question | Read |
|---|---|
| *What did we charge for this line?* | `order_line.line_amount_minor` |
| *Why that amount?* | `price_source_kind` + `price_source_id` (+ `list_price_id`, `percent_off`) |
| *What was the agreed price for product X on 3 March?* | the `customer_price` / `price_list_item` intervals |
| *What is the price today?* | the same intervals, resolved at `$now` |

---

## Step 6 — The sealed report

```sql
CREATE TABLE billing_report (
  id           TEXT PRIMARY KEY,
  period_start TEXT NOT NULL,
  period_end   TEXT NOT NULL,      -- exclusive, like every interval here
  status       TEXT NOT NULL DEFAULT 'draft',
  sealed_at    TEXT,
  total_minor  INTEGER NOT NULL DEFAULT 0,
  currency     TEXT NOT NULL,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)

CREATE TABLE billing_report_line (
  id                TEXT PRIMARY KEY,
  billing_report_id TEXT NOT NULL,
  order_line_id     TEXT,          -- NULL only for an adjustment line
  adjusts_line_id   TEXT,          -- set on an adjustment; points at the corrected line
  amount_minor      INTEGER NOT NULL,
  reason            TEXT,
  created_at        TEXT NOT NULL,
  updated_at        TEXT NOT NULL
)
```

Bind an FSM so the seal is enforced by the platform rather than by convention:

```yaml
action: state_machines.register
input:
  kind: state_machine
  track: billing
  name: BillingReport.status
  objectType: billing_report
  property: status
  initial: draft
  states:
    draft:  { label: Draft }
    sealed: { label: Sealed, terminal: true }
  transitions:
    - { via: billing_report.seal, from: [draft], to: sealed }
```

`sealed` is **terminal**, so there is no transition out. Declare no
`billing_report_line.update` and no `billing_report_line.delete` at all — the
lines of a sealed report are not editable by any path, which is a stronger
guarantee than an action that checks a flag.

`billing_report.seal` stamps `sealed_at` and writes `total_minor` as the integer
sum of its lines. After that the report reproduces byte for byte, forever,
because it stores its own numbers rather than recomputing them.

### Corrections go forward

June was wrong. You find out on 12 July. **Do not touch June.**

```
billing_report_line {
  billing_report_id: <JULY's report>,
  order_line_id:     null,
  adjusts_line_id:   <the June line that was wrong>,
  amount_minor:      -4250,
  reason:            "June: quantity 3 invoiced, 2 delivered"
}
```

June still reconciles against what was sent in June. July carries a visible,
explained adjustment. Both months are reproducible, and the correction is
legible **as** a correction — which is what a reader actually wants to see.
Rewriting June makes the error invisible and makes both months untrustworthy,
because a report that can change retroactively is a report nobody can cite.

The same rule covers every artefact you have handed to somebody as a fact: a
sent invoice, a filed VAT return, a published commission statement, a signed
proposal.

---

## What you end up with

- Any historical invoice reproduces exactly, from stored numbers.
- Any charge traces to the agreement that produced it, by id.
- A price change is a scheduled row, not a deployment.
- Nothing in the money path is a float; one rounding step, written down.
- Corrections are visible instead of silent.
- A warehouse can consume the price tables as type-2 dimensions with no ETL
  reconstruction.

None of that is retrofittable. The history you did not write is not recoverable.
