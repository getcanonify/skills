# Pricing, end to end

Obligation 4 as one build: effective-dated list and customer prices, resolution
on the order's date, a settlement that snapshots, and a sealed report. Products
have a list price; some customers have a negotiated override; order lines must
reproduce forever; month end hands invoicing a sealed report.

## 1 — Tables

```sql
CREATE TABLE price_list_item (
  id TEXT PRIMARY KEY, product_id TEXT NOT NULL,
  amount_minor INTEGER NOT NULL,          -- øre; never REAL (C10-float-money)
  valid_from TEXT NOT NULL, valid_to TEXT, -- [from, to); NULL = still in force
  note TEXT,                              -- why it changed
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
CREATE TABLE customer_price (
  id TEXT PRIMARY KEY, customer_id TEXT NOT NULL, product_id TEXT NOT NULL,
  amount_minor INTEGER NOT NULL,          -- 0 = deliberately free; no row = no agreement
  valid_from TEXT NOT NULL, valid_to TEXT, agreement_ref TEXT,
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
CREATE TABLE order_line (
  id TEXT PRIMARY KEY, order_id TEXT NOT NULL, product_id TEXT NOT NULL,
  quantity INTEGER NOT NULL,
  unit_amount_minor INTEGER NOT NULL,     -- THE SNAPSHOT, frozen at settlement
  line_amount_minor INTEGER NOT NULL,
  price_source_kind TEXT NOT NULL,        -- THE BACK-REFERENCE: customer_price | price_list_item
  price_source_id TEXT NOT NULL,
  priced_at TEXT NOT NULL,                -- the date resolution ran against
  created_at TEXT NOT NULL, updated_at TEXT NOT NULL
);
```

`customer_order` carries `placed_at` — **the** resolution date. A percentage
agreement adds a `percent_off INTEGER` (basis points, `1200` = 12%) and, on the
line, the list-price row's id beside the discount row's.

## 2 — Refine: temporal, anchored, money

```yaml
action: catalog.refine_object_type
input:
  name: customer_price
  description: >-
    Negotiated price for one (customer, product), effective-dated and
    append-only. amount_minor 0 means deliberately free; a missing row means
    no agreement — fall through to the list price.
  temporal: { from: valid_from, to: valid_to, key: [customer_id, product_id] }
  links:
    customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
    product:  { target: product,  cardinality: one, via: product_id,  fk_on: source }
  properties:
    amount_minor: { kind: mapped, column: amount_minor, type: money, currency: DKK }
```

`price_list_item` is the same with `key: [product_id]`. `order_line` links
upward to its order and product, declares both amounts `type: money`, narrows
`price_source_kind` to an enum, and states the rounding policy in its
description: *"Half-up to whole øre, once per line; the order total is the
integer sum of line amounts and is never rounded again."*

## 3 — Repricing

`temporal` generates `price_list_item.set`: it closes the product's open period
at the new `valid_from`, inserts the new row, and refuses any overlap
(`temporal_overlap`). `amount_minor` is never written twice; `valid_from` is an
input, so a price agreed today can take effect on 1 April with nothing to do on
the day. Hand-declare `price_list_item.set` only to add domain guards — it then
replaces the generated verb.

## 4 — Resolve and settle in one action

The customer override wins, else the list price, both read **as of the order's
`placed_at`**; no list price in effect refuses the line (`read` → `NOT_FOUND`)
rather than defaulting to 0 or to today's price. Two inserts with opposite
`when` guards are the if/else:

```json
{
  "steps": [
    { "op": "read", "object": "customer_order", "where": { "id": "$input.order_id" }, "as": "order" },
    { "op": "read_optional", "object": "customer_price", "as": "cp",
      "where": { "customer_id": "$order.customer_id", "product_id": "$input.product_id", "$as_of": "$order.placed_at" } },
    { "op": "read", "object": "price_list_item", "as": "list",
      "where": { "product_id": "$input.product_id", "$as_of": "$order.placed_at" } },
    { "op": "insert", "object": "order_line", "when": { "ref": "$cp.id", "present": true },
      "values": { "order_id": "$input.order_id", "product_id": "$input.product_id", "quantity": "$input.quantity",
        "unit_amount_minor": "$cp.amount_minor",
        "line_amount_minor": { "op": "*", "left": "$cp.amount_minor", "right": "$input.quantity" },
        "price_source_kind": "customer_price", "price_source_id": "$cp.id", "priced_at": "$order.placed_at" } },
    { "op": "insert", "object": "order_line", "when": { "ref": "$cp.id", "absent": true },
      "values": { "order_id": "$input.order_id", "product_id": "$input.product_id", "quantity": "$input.quantity",
        "unit_amount_minor": "$list.amount_minor",
        "line_amount_minor": { "op": "*", "left": "$list.amount_minor", "right": "$input.quantity" },
        "price_source_kind": "price_list_item", "price_source_id": "$list.id", "priced_at": "$order.placed_at" } }
  ]
}
```

A percentage agreement computes `round(list × (10000 − bp) / 10000)` in the same
insert — integers until the one rounding step. After this the line is a fact:
reprice tomorrow, renegotiate next month, and it still says what it charged and
which period justified it, because that period was closed, not overwritten.

| Question | Read |
|---|---|
| What did we charge? | `order_line.line_amount_minor` |
| Why that amount? | `price_source_kind` + `price_source_id` |
| What was agreed for product X on 3 March? | the price rows, `--as-of 2026-03-03` |

## 5 — The sealed report

`billing_report (period_start, period_end, status, sealed_at, total_minor,
currency, …)` and `billing_report_line (billing_report_id, order_line_id,
adjusts_line_id, amount_minor, reason, …)`, both anchored upward. Bind
`BillingReport.status` (`draft → sealed` via `billing_report.seal`, `sealed`
terminal); the seal stamps `sealed_at` and writes `total_minor` as the integer
sum of the lines. Declare no update or delete for lines at all — a stronger
guarantee than an action checking a flag.

June was wrong and you find out on 12 July: do not touch June. Add a line to
**July's** report with `order_line_id: null`, `adjusts_line_id: <the June
line>`, `amount_minor: -4250`, `reason: "June: 3 invoiced, 2 delivered"`. Both
months reproduce, and the correction reads as a correction.
