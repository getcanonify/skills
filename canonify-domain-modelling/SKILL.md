---
name: canonify-domain-modelling
description: The four obligations every Canonify domain model must meet — reuse the canonical object instead of retyping it, anchor every child to its owner, reuse the platform primitive before building one, and model time and money as effective-dated append-only rows with snapshotted settlements. States each as a rule with a worked before/after, and names the coherence-audit code (C1-shadow-entity, C2-unanchored, C10-float-money, C13-primitive-dup) that fires when you break it. Read after the base `canonify` skill and before you declare your first table.
version: 1.0.0
---

# Canonify Domain Modelling — the four obligations

The other skills teach you the **mechanics**: how to declare a link, what
`cardinality` and `on_delete` mean, which column `via` points at. They are
complete and correct, and they are not enough. Mechanics tell you what is
*possible*; they never told you what is *obligatory*.

This skill is the obligations. Each one is a rule, not a tip. Each one has a
worked before/after taken from a real 54-object-type Canon that broke it. Each
one names the coherence-audit code that fires when you break it, so a
diagnostic you get back points at the paragraph that explains it.

| # | Obligation | Audit code |
|---|---|---|
| 1 | **Reuse the canonical object.** Never re-declare an entity that already has an ObjectType. | `C1-shadow-entity` |
| 2 | **Anchor every child.** An FK column without an upward link is a defect. | `C2-unanchored` |
| 3 | **Reuse the platform primitive.** Check before you model a capability. | `C13-primitive-dup` |
| 4 | **Effective-date time, snapshot money.** Prices are dated append-only rows; amounts are integer minor units. | `C10-float-money` |

Read all four before your first `schema.propose_migration`. Three of them are
almost impossible to retrofit — by the time a `proposal` table has ten thousand
rows carrying a retyped customer name, nobody knows which of the four spellings
of "Nordic Foods A/S" is the same company.

> **Where these came from.** Every example below is from a structural sweep of a
> real production Canon: 54 ObjectTypes, 73 actions, 66 views, 57 tables. It was
> valid — it decoded, it applied, it ran. It was also wrong in all four ways, and
> the skills at the time contained roughly 3,300 lines of accurate mechanics and
> not one sentence saying so.

---

## Obligation 1 — Reuse the canonical object

> **Rule.** A property whose value names a real-world entity that already has
> an ObjectType is a defect. Declare a link to that ObjectType and expose the
> display value as a `kind: "derived"` property through the link. Never a
> mapped text column.

This is the one that fails silently, because a `TEXT NOT NULL` column holding a
customer's name is perfectly valid. It decodes, it applies, it renders. It is
also a second, unreconciled model of a thing you already model.

The pressure that produces it is always local and always reasonable: *this
proposal page needs to show the customer's name, address and CVR*. Adding three
columns takes one migration; declaring a link and three derived properties takes
the same migration plus a refinement, and feels like ceremony. It is not
ceremony. It is the difference between a record that *is about* a customer and a
record that *mentions* one.

### The before — a shadow entity

A real `proposal` ObjectType, backed by these columns:

```sql
CREATE TABLE proposal (
  id                TEXT PRIMARY KEY,
  customer_name     TEXT NOT NULL,   -- display.title is "{customer_name}"
  customer_address  TEXT,
  customer_cvr      TEXT,            -- duplicates organization's CVR anchor
  contact_person    TEXT,
  contact_email     TEXT,            -- duplicates contact
  account_owner     TEXT,            -- duplicates customer.account_owner
  deal_id           TEXT,            -- an FK column with no link over it
  ...
)
```

`customer`, `organization` and `contact` all exist as ObjectTypes in the same
Canon. The proposal does not point at any of them. It retypes them.

What that costs, concretely:

- **Identity is a free-text string.** `display.title` resolves to
  `{customer_name}`, so the record's own name is whatever a salesperson typed.
  "Nordic Foods A/S", "Nordic Foods", "nordic foods a/s" are three customers.
- **Nothing propagates.** The customer moves office; every proposal keeps the
  old address forever. There is no update path, because there is no join.
- **You cannot navigate.** `canon objects navigate customer cus_… proposals`
  returns nothing, because no link exists. Neither does a record page: the
  renderer has no way to scope proposals to the customer whose page it is.
- **You cannot aggregate.** "Total proposed value per customer" is a fuzzy
  string match, not a query.
- **Deal → proposal → contract is three disconnected islands**, each holding
  its own spelling of the same company.

The coherence audit reports this as **`C1-shadow-entity`**, at `error`
severity, once per offending column.

### The after — a link plus derived properties

Same Canon, different object. `customer` gets its organisation *right*:

```yaml
action: catalog.refine_object_type
input:
  name: customer
  links:
    organization: { target: organization, cardinality: one, via: organization_id, fk_on: source }
  properties:
    organization_name:
      kind: derived
      type: string
      expression: SELECT name FROM organization WHERE organization.id = customer.organization_id
```

A `derived` property's `expression` is a **scalar SQL expression** — usually a
correlated subquery through the FK column the link declares. The platform
projects it into the ObjectType's read view as `(<expression>) AS
"<property>"`, so it is queryable, filterable and renderable exactly like a
mapped column. It is read-only: no handler can write it, which is the point.

One link, one derived property, and the name is now **read through** the
organisation rather than copied from it. Rename the organisation once and every
customer, proposal, contract and report shows the new name on the next read.
There is exactly one row in the system that knows what this company is called.

The same shape fixes `proposal`. Drop the six retyped columns, keep a
`customer_id` FK, and declare:

```yaml
action: catalog.refine_object_type
input:
  name: proposal
  links:
    customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
    deal:     { target: deal,     cardinality: one, via: deal_id,     fk_on: source }
  properties:
    customer_name:
      kind: derived
      type: string
      expression: SELECT name FROM customer WHERE customer.id = proposal.customer_id
  display:
    title: { template: '{customer_name} — {reference}' }
```

`display.title` still renders the customer's name. It is now *the* customer's
name.

The title strategy has three shapes — `{ property: … }`, `{ template: … }`, and
`{ derived: { via: <link-name> } }`. That third shape is the strongest form of
this obligation: it titles the record **through the link**, by resolving the
target's own title.

```yaml
  display:
    title: { derived: { via: customer } }
```

Nothing is copied, nothing is interpolated, and the proposal's identity is the
customer's identity — which is what it always was.

> **Same bundle. Same concept. Opposite outcomes.** `customer.organization_name`
> is derived through a link; `proposal.customer_name` is a mapped `TEXT NOT NULL`
> that titles the record. One author knew the obligation and one did not, and
> nothing in the platform's documentation told them apart. That is what this
> skill exists to end.

### The test to apply

Before you add any column, ask: **does this value name something that has, or
should have, its own ObjectType?**

| The column holds… | Do this |
|---|---|
| A person, company, product, place, or account | Link to it. Derive the display value. |
| An identifier belonging to another entity (CVR, VAT no., email of a known contact) | Link. That identifier is the *other* entity's property, not yours. |
| Something true only of THIS record at THIS moment (a quantity, a note, a status) | Mapped column. Correct. |
| A value that was true at settlement time and must never change afterwards | Mapped column **plus** a link to the row it came from. See Obligation 4. |

The last row is the one real exception, and it is not a loophole — a snapshot is
a deliberate, documented copy that carries a reference back to its source. A
shadow entity is an accidental copy that carries nothing.

**When you cannot fix it in one step**, the sequence is: add the FK column and a
link → backfill `customer_id` by matching the text (this is the painful part,
and it gets more painful every day you wait) → add the derived property → switch
`display.title` and every view column to the derived property → drop the text
column last. Do not stop after step three: a shadow column left "for safety"
diverges immediately.

---

## Obligation 2 — Anchor every child

> **Rule.** If an ObjectType has an FK column naming another ObjectType, it MUST
> declare the upward link. A downward `cardinality: many` link on the parent is
> not a substitute, and neither is the column existing.

The column carries the value. The **link** carries the meaning. Without the
link, the platform does not know the column is a reference, so:

- you cannot navigate the child to its owner — no `canon objects navigate`, no
  breadcrumb, no "back to the deal" affordance;
- a record page cannot scope a child table to the record it is showing. The
  renderer needs the link to build the filter, so the child table either shows
  every row in the org or cannot be placed on the page at all;
- `on_delete` has nothing to attach to, so deleting the parent leaves orphans
  instead of restricting, cascading, or nulling;
- typed reads cannot traverse, so a ViewSpec column ref like
  `customer.organization.name` does not resolve.

In the audited Canon, **24 of 54 ObjectTypes** had at least one FK column
naming a declared ObjectType and no link over it — including `deal`, `contact`,
`activity`, `employee`, `customer_order`, and all six `proposal_*` children.
For the record: **0 of 57 tables** carried a SQL `REFERENCES` clause either.
Nothing anywhere in the system knew those columns pointed at anything.

`proposal` is the sharpest case. It declared **five** links — `comments`,
`line_items`, `participants`, `shares`, `views` — and **nothing upward**. It
knew all of its children and none of its parents. That is why deal → proposal →
contract was three islands: every hop downward existed and every hop upward was
missing.

### Before

```yaml
# proposal — five downward links, no upward link. deal_id is inert.
links:
  comments:     { target: proposal_comment,     cardinality: many, via: proposal_id }
  line_items:   { target: proposal_line_item,   cardinality: many, via: proposal_id }
  participants: { target: proposal_participant, cardinality: many, via: proposal_id }
```

### After

```yaml
links:
  # Upward — the anchor. fk_on: 'source' says the FK column lives on THIS object.
  deal:     { target: deal,     cardinality: one, via: deal_id,     fk_on: source }
  customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
  # Downward — unchanged.
  comments:     { target: proposal_comment,     cardinality: many, via: proposal_id, on_delete: cascade }
  line_items:   { target: proposal_line_item,   cardinality: many, via: proposal_id, on_delete: cascade }
  participants: { target: proposal_participant, cardinality: many, via: proposal_id, on_delete: cascade }
```

The direction is the whole point. On a `cardinality: one` upward link `via`
names the FK column **on this object**, so mark it `fk_on: source`. On a
`cardinality: many` downward link `via` names the FK column **on the target**.
Getting this backwards is the most common way an "anchored" object is still
unanchored.

The coherence audit reports the missing anchor as **`C2-unanchored`**, at
`error` severity.

> **The render-side symptom of the same defect.** Catalog validation already
> emits an advisory, `elegance_unlinked_child_table`, when a view puts a child
> table on a record page with no link to filter it by. In the audited Canon it
> fired **eighteen times** and nobody acted on it, because an advisory that
> nothing blocks on is indistinguishable from silence. It is not a rendering
> nit. It is `C2-unanchored` showing up at the surface. Fix the link and the
> advisory disappears with it.

**Checking your own Canon:** for every table, list the columns ending in `_id`,
and check each one against `canon objects list-types`. If the prefix names an
ObjectType, that object owes an upward link.

```sh
canon schema describe --object proposal   # columns + declared links, side by side
canon objects list-types                  # the names an FK prefix must match
```

---

## Obligation 3 — Reuse the platform primitive

> **Rule.** Before modelling a capability, check whether Canonify already ships
> it. If it does, compose it. Building a second implementation of a platform
> primitive is a defect even when your version works.

This one has a procedure, because an obligation with no procedure is a wish.

### The procedure — three commands, before any migration

```sh
canon objects list-types          # every ObjectType, platform + yours
canon actions list --json         # every action, platform + yours
canon schema describe             # the data-model overview
```

Run these **in a fresh org, before you declare anything**. What comes back on an
empty Canon is the platform surface — everything you get for free. Read it once
and keep the list. Over REST that is `GET /v1/actions`
and `GET /v1/schema_describe`; over MCP, the `actions.list` tool. All three
return the same catalog.

Then, for the capability you are about to build, search that list for its nouns
before you write its columns. `canon actions describe <name> --json` prints the
full declaration of anything that looks close.

### The before — ten object types for something the platform ships

The audited Canon built its own e-signature: ten ObjectTypes across
`proposal*`, `document`, and `document_signature`, with **12 signature columns
on the `proposal` table alone** (`signature_url`, `signed_at`, `signer_name`,
`signer_ip`, …). The platform ships e-signature as a first-class primitive —
`signature_request` for a single signer, `signature_envelope` for ordered
multi-signer flows, `signature` as the immutable signed record, plus
`signature.verify` and `signature.certificate`.

The bundle referenced `signature_envelope` **zero** times.

What the hand-rolled version did not get, and could not cheaply add: mandatory
signing consent, `link_token` redaction from the audit trail, server-derived
signing evidence (the platform derives the IP and user agent rather than
trusting a client field), a verifiable certificate, propagating terminal
lifecycles (a decline ends the envelope; a void clears the countersign queue;
completion is automatic), and external-signer link redemption. Twelve columns
bought a status string.

The same Canon also re-implemented file storage: seven url/type column pairs on
`proposal` and six on `proposal_template`, where a property of `type: 'file'`
stores the bytes and carries the content type already.

The coherence audit reports both as **`C13-primitive-dup`**, at `error`
severity, when a table carries three or more columns matching a known
primitive's shape.

### The after — compose it

```yaml
# The proposal keeps ONE link to the envelope. The envelope owns signing.
links:
  signature_envelope: { target: signature_envelope, cardinality: one, via: signature_envelope_id, fk_on: source }
properties:
  signing_status:
    kind: derived
    type: string
    expression: SELECT status FROM signature_envelope WHERE signature_envelope.id = proposal.signature_envelope_id
```

Twelve columns become one FK and one derived property, and you inherit consent,
evidence, redaction, certificates and the lifecycle. The full invoke-by-invoke
surface is in
[`canonify-objects-actions` → E-signature actions](../canonify-objects-actions/resources/esign-actions.md).

### Capabilities to check for before building

Signing and files are the two the audit detects mechanically, but the habit is
general. Before you build any of these, look first:

| You are about to build… | Look for |
|---|---|
| Signing, approval-with-signature, countersigning | `signature_request`, `signature_envelope`, `signature`, `member_signature` |
| File upload, attachments, document storage | a property of `type: 'file'` |
| An approval gate on an operation | `governance.requires_approval` / `requires_approval_when` on the ActionType |
| An audit log, a "who changed what" table | `_audit_event` — every invocation already writes one. `canon audit` reads it. |
| A retry queue, a "send later" table, webhook dispatch | the outbox — actions enqueued by a handler or an FSM hook |
| A status column plus hand-written legality checks | a StateMachine bound to an enum property |
| A "has this already run?" guard table | the ActionType's `idempotency` posture |

If your table has a column that duplicates something in that right-hand column,
you are maintaining a worse copy of a thing that is already tested, audited and
supported.

---

## Obligation 4 — Time and money

> **Rule.** A value that changes over time and matters financially is an
> **effective-dated, append-only row** — never a mutable column. Money is
> **integer minor units** — never a float. A settled transaction **snapshots**
> the value it used and **references** the row it resolved from.

This is the largest obligation and the one with the least intuition behind it,
because the wrong version works perfectly right up until the first month-end
close, and then it is unfixable — the history you needed was never written.

The worked example is pricing, end to end. Everything here generalises to any
value that is agreed on one date and applied on another: rates, fees, salaries,
commission percentages, tax rates, SLA thresholds.

### 4.1 Effective-dated rows, append-only

> **Never update a price. Never delete a price.** Changing a price means
> **closing** the current row and **inserting** a new one.

```sql
CREATE TABLE price_list_item (
  id           TEXT PRIMARY KEY,
  product_id   TEXT NOT NULL,
  amount_minor INTEGER NOT NULL,   -- øre / cents. See 4.6.
  currency     TEXT NOT NULL,
  valid_from   TEXT NOT NULL,      -- ISO 8601 UTC, INCLUSIVE
  valid_to     TEXT NOT NULL,      -- ISO 8601 UTC, EXCLUSIVE; OPEN_END = still in force
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)
```

Two conventions carry the whole model, and both are worth stating in the
ObjectType description so nobody has to infer them:

> **Half-open intervals.** `valid_from` is inclusive, `valid_to` is
> **exclusive**. The in-effect test is `valid_from <= :at AND :at < valid_to`.
> Closing a row then means setting its `valid_to` to *exactly* the successor's
> `valid_from` — no date arithmetic, no off-by-one, and provably no gap and no
> overlap. Inclusive-on-both-ends forces you to subtract a day, and every
> subtract-a-day is a bug waiting for a timezone.

> **An open-end sentinel, not `NULL`.** Write `'9999-12-31T00:00:00.000Z'` for
> "still in force". Comparisons stay plain (no `coalesce`), and "the currently
> open row" becomes an equality match you can put straight into a handler
> `where` — which a nullable column cannot be, since `= NULL` matches nothing.
> `NULL` also overloads one column with *open-ended*, *unknown* and *not
> applicable* at once, which is exactly the collapse §4.3 forbids.

Timestamps everywhere, per the platform convention — ISO 8601 UTC `TEXT`, never
an epoch integer. A date-only agreement is just midnight UTC.

A price change effective 2026-04-01 is **two writes in one action**, not one
update:

```json
{
  "name": "price_list_item.reprice",
  "input": {
    "product_id":   { "type": "string" },
    "amount_minor": { "type": "number" },
    "valid_from":   { "type": "string", "format": "iso8601" }
  },
  "handler": {
    "steps": [
      { "op": "read",   "object": "price_list_item",
        "where": { "product_id": "$input.product_id", "valid_to": "9999-12-31T00:00:00.000Z" },
        "as": "current" },
      { "op": "update", "object": "price_list_item",
        "where": { "id": "$current.id" },
        "set":   { "valid_to": "$input.valid_from" } },
      { "op": "insert", "object": "price_list_item",
        "values": { "product_id": "$input.product_id", "amount_minor": "$input.amount_minor",
                    "currency": "$current.currency", "valid_from": "$input.valid_from",
                    "valid_to": "9999-12-31T00:00:00.000Z" },
        "as": "next" }
    ],
    "returns": { "closed_id": "$current.id", "opened_id": "$next.id" }
  }
}
```

The one `update` here is not a mutation of the value — it is the closing of an
interval. The **amount is never touched**. That is the invariant.

Three things fall out of this one rule, and you get all three for free:

1. **Full reconstructable history.** "What did we charge this customer in
   February?" is answerable in March, and still answerable in three years, from
   the rows themselves — no audit-log archaeology, no backups.
2. **Scheduling.** A price agreed today and effective 1 April is just a row with
   `valid_from: '2026-04-01T00:00:00.000Z'`. Nobody has to remember to change
   anything on the day, no cron job flips a column at midnight, and nobody can
   apply it early by accident. A mutable column cannot express "true later" at
   all.
3. **It is already the shape a warehouse wants.** Effective-dated rows are a
   type-2 slowly-changing dimension. ETL consumes them as-is. A mutable price
   column has to be reconstructed from change logs, badly, forever.

**Intervals must not overlap for the same scope.** Close first, then insert, in
**one action** so both writes share a transaction — a partial apply that closes
the old row without opening the new one leaves the product unpriced. Because the
interval is half-open, adjacency is just `closed.valid_to == next.valid_from`,
which is what the handler above writes.

### 4.2 Resolution by specificity

Prices exist at more than one scope: a list price for everyone, and an override
negotiated with one customer. **The most specific row in effect on the relevant
date wins.**

State that as a **two-case lookup**, not as a ranking engine:

```
resolve(product, customer, at):
  1. the customer_price row for (customer, product) where
     valid_from <= at AND at < valid_to      → if found, use it
  2. otherwise the price_list_item row for product, same window
```

Two scopes is two cases. Do not build a priority-number column, a scope-ranking
table, or a rules engine to express a rule that fits in four lines — you will
maintain the engine forever and the fifth scope never arrives. If a third scope
genuinely does arrive (a contract price above a customer price), it is a third
case in the same ordered lookup, still not an engine.

The date you resolve against is **the date the transaction happened**, not
today. Resolving against `$now` when re-running last month's billing is how you
silently rewrite February.

### 4.3 A row with `0` is NOT the same as no row

> **These must never collapse.** `0` means *deliberately free* — someone agreed
> this customer pays nothing for this product, on this date, for this reason.
> **Absent** means *no agreement exists* — fall through to the next case.

If your resolution code treats a missing row as zero, or a zero as missing, you
have two bugs waiting: a customer with a negotiated free line gets billed at
list price, or a customer with no agreement gets billed nothing and nobody
notices until the quarter closes.

Concretely, this means:

- The amount column is `INTEGER NOT NULL`. Not nullable. If there is no
  agreement, there is **no row** — you never encode absence as a NULL amount.
- Resolution asks "did I find a row?", never "is the amount truthy?".
- A negotiated zero is inserted like any other price: a row, with
  `amount_minor: 0` and its own `valid_from`.

The same discipline is why §4.1 uses an explicit open-end sentinel rather than a
nullable `valid_to`. `NULL` is the classic overloaded value — *open-ended*,
*unknown* and *not applicable* all look identical to a reader and to a query.
Give each meaning its own representation, or do not admit the meaning.

### 4.4 Snapshot the resolved value onto the transaction line

> **Rule.** At the moment a transaction settles, write the resolved amount
> **onto the line**, together with a **reference to the row it resolved from**.

```sql
CREATE TABLE order_line (
  id                    TEXT PRIMARY KEY,
  order_id              TEXT NOT NULL,
  product_id            TEXT NOT NULL,
  quantity              INTEGER NOT NULL,
  -- the snapshot: what we actually charged, frozen at settlement
  unit_amount_minor     INTEGER NOT NULL,
  currency              TEXT NOT NULL,
  -- the back-reference: which agreement produced it
  price_source_kind     TEXT NOT NULL,   -- 'customer_price' | 'price_list_item'
  price_source_id       TEXT NOT NULL,
  priced_at             TEXT NOT NULL,   -- the date resolution ran against
  created_at            TEXT NOT NULL,
  updated_at            TEXT NOT NULL
)
```

Both halves are mandatory, and they answer **two different questions**:

| Question | Answered by | Why not the other one |
|---|---|---|
| *What did we charge?* | the **snapshot** (`unit_amount_minor` on the line) | A live re-resolve gives a different answer after any repricing. An invoice is a fact, not a query. |
| *What was the agreed price on 3 March?* | the **dated rows** (`price_list_item` / `customer_price`) | The snapshot only knows the one line it is on. It cannot tell you the price of something nobody bought. |

With both, a billed line traces back to the exact agreement that produced it:
"we charged 4,250 øre because customer price `cp_01H…`, valid from 1 Feb, said
so." And renegotiating in March **cannot rewrite February**, because February's
lines carry their own amounts and point at the rows that were in effect then —
rows which are still there, closed, not deleted.

Note this is the documented exception to Obligation 1. `unit_amount_minor` is a
copy of a value that lives elsewhere, and it is correct precisely because it is
deliberate, immutable, and carries `price_source_id` back to its origin. A
shadow entity is a copy that points at nothing.

### 4.5 Percentage discounts make list-price history load-bearing

A customer agreement that says *"12% off list"* is **not** a price. It is a
function of a price, and it is only meaningful against the list price **on the
day the transaction happened**.

So:

- The discount row is effective-dated exactly like a price row
  (`percent_off`, `valid_from`, `valid_to`).
- Resolution is two lookups, not one: find the list price in effect on the
  transaction date, then find the discount in effect on the same date, then
  apply.
- **The list price's history is now load-bearing.** If you had "improved"
  `price_list_item` by updating the amount in place, every historical percentage
  discount silently recomputes against today's list price, and last year's
  invoices no longer reproduce. Percentage agreements are the reason
  Obligation 4.1 is not optional.
- Snapshot **the computed amount**, per 4.4 — plus the discount row's id
  alongside the list-price row's id, so the line records both inputs.

If you can, prefer an absolute agreed amount over a percentage: it needs one
lookup and no arithmetic to reproduce. But when the business genuinely agreed a
percentage, model the percentage — do not silently freeze it into an amount and
lose the fact that it floats.

### 4.6 Money is integer minor units

> **Rule.** Store money as `INTEGER` in the currency's minor unit — øre, cents,
> pence. Never `REAL`. Never a float anywhere in the path.

Binary floating point cannot represent `0.1`. Totals drift at the minor unit,
and they drift *further* the more lines you sum, so the error is largest exactly
where it is most visible: a big invoice, a month-end reconciliation, a VAT
return. The audited Canon stored `order_line.unit_price`, `order_line.amount`,
`product.default_price` and three `proposal` price columns as SQLite `REAL`,
while its own roadmap made *"one month's report reconciles against the same
month done the old way"* the acceptance criterion for going live. Float money
produces øre-level mismatches at precisely the moment reconciliation is the
gate.

```sql
-- Wrong
amount REAL NOT NULL

-- Right
amount_minor INTEGER NOT NULL,
currency     TEXT NOT NULL
```

Declare the property with its currency so it renders as an amount rather than a
count:

```yaml
properties:
  amount_minor:
    kind: mapped
    column: amount_minor
    type: number
    currency: DKK
```

The coherence audit reports a `REAL` money column as **`C10-float-money`**, at
`error` severity.

Two habits that go with it:

- **Carry the currency on the row**, next to the amount. An integer with no
  currency is not money. Never mix currencies inside one sum without an explicit
  conversion, itself dated.
- **Never divide into a float on the way through.** Compute in integers,
  round once, at a defined point (below).

### 4.7 State the rounding policy explicitly

Percentages and per-unit rates produce fractions of a minor unit. Somebody has
to decide what happens to them. If you do not decide, each code path decides
differently and finance finds out at month end.

Write the policy down, in the ObjectType description and in the action that
applies it, and make it say three things:

1. **Where** rounding is applied — per line, or on the invoice total? (Per line
   is the usual choice: it makes each line independently reproducible, and the
   total is then just a sum of integers with no further rounding.)
2. **To how many places** — normally zero decimals of the minor unit, i.e. a
   whole øre.
3. **Which direction** — half-up, half-even, or always-down. Half-up is the
   common commercial default; half-even is common in finance. Either is fine.
   *Unstated* is not.

> Example policy, stated: *"Line amounts are computed as
> `round_half_up(unit_amount_minor × quantity × (1 − percent_off/100))` to whole
> øre, rounded once per line. The invoice total is the integer sum of rounded
> line amounts; the total is never rounded again."*

That sentence is the difference between a 3-øre discrepancy you can explain and
one you cannot.

### 4.8 Periodic reports are sealed objects, not live queries

> **Rule.** What you handed to invoicing for June is a **stored object** with
> stored lines — not a saved query that recomputes.

A live query is a different answer every time you run it. A late order, a
corrected quantity, a repriced product, a customer merged into another — any of
these silently changes "June" after June was already invoiced, and the report
you emailed on 1 July can no longer be reproduced. Nobody notices until an
auditor asks, and by then the difference has no explanation.

So seal it:

```sql
CREATE TABLE billing_report (
  id           TEXT PRIMARY KEY,
  period_start TEXT NOT NULL,
  period_end   TEXT NOT NULL,
  status       TEXT NOT NULL,   -- draft | sealed  (bind a StateMachine)
  sealed_at    TEXT,
  total_minor  INTEGER NOT NULL,
  currency     TEXT NOT NULL,
  created_at   TEXT NOT NULL,
  updated_at   TEXT NOT NULL
)

CREATE TABLE billing_report_line (
  id                TEXT PRIMARY KEY,
  billing_report_id TEXT NOT NULL,   -- link it upward. Obligation 2.
  order_line_id     TEXT NOT NULL,   -- link it upward too
  amount_minor      INTEGER NOT NULL,
  created_at        TEXT NOT NULL,
  updated_at        TEXT NOT NULL
)
```

`draft → sealed` is a StateMachine transition, and **a sealed report is
immutable**: no action may update its lines or its total. Bind the FSM, mark
`sealed` terminal, and let the transition guard be the enforcement.

**Corrections land forward, never backward.** Something wrong in June that you
discover in July becomes an **adjustment line on July's report**, carrying a
reference to the June line it corrects. History stays reproducible and the
correction is visible as a correction — which is what a reader actually wants to
see. Rewriting June makes the error invisible and makes both months unreliable.

The same rule covers anything you have handed to someone as a fact: a sent
invoice, a filed VAT return, a published commission statement, a signed
proposal. Once it left the building, it is a stored object.

---

## The checklist

Run this before every `schema.propose_migration`, and again before you declare
the ObjectType that refines it.

- [ ] Does any column name an entity that already has an ObjectType? → link + derived property, not text (`C1-shadow-entity`)
- [ ] Does every `*_id` column have a declared upward link with `fk_on: source`? (`C2-unanchored`)
- [ ] Did I run `canon objects list-types` and `canon actions list --json` before modelling this capability? (`C13-primitive-dup`)
- [ ] Is every money column `INTEGER` minor units with a currency beside it? (`C10-float-money`)
- [ ] Does every value that can change over time have `valid_from` / `valid_to`, and is it written append-only?
- [ ] Does every settled line carry both the snapshot **and** the id of the row it resolved from?
- [ ] Is the rounding policy written down — where, how many places, which direction?
- [ ] Are `0` and *no row* distinguishable everywhere they are read?
- [ ] Is every periodic report a stored, sealable object rather than a live query?

## See also

- [`canonify`](../canonify/SKILL.md) — the base manual: the four declaration
  verbs, the runtime frame, idempotency, audit/outbox.
- [`canonify-objects-actions`](../canonify-objects-actions/SKILL.md) — the
  mechanics these obligations constrain: property kinds
  ([object-property-types.md](../canonify-objects-actions/resources/object-property-types.md)),
  links, `cardinality`, `via`, `fk_on`, `on_delete`
  ([object-links-and-cardinality.md](../canonify-objects-actions/resources/object-links-and-cardinality.md)),
  and the built-in e-signature surface
  ([esign-actions.md](../canonify-objects-actions/resources/esign-actions.md)).
- [`canonify-app-authoring`](../canonify-app-authoring/SKILL.md) — the
  five-chapter authoring loop these obligations apply to, chapter by chapter.
- [`canonify-app-lifecycle`](../canonify-app-lifecycle/SKILL.md) — the same
  audited canon one level up: how many apps to build at once, what "1.0" has to
  mean, and the surface-level coherence codes (`C5`–`C9`). Obligation 4 is what
  makes its closed-period replay possible at all.
- [`canonify-viewspecs`](../canonify-viewspecs/SKILL.md) — why an anchored child
  is what lets a record page scope a child table at all.
- [resources/pricing-worked-example.md](resources/pricing-worked-example.md) —
  the whole of Obligation 4 as one continuous build: tables, refinements,
  resolution action, settlement action, sealing FSM.
- [resources/anti-patterns.md](resources/anti-patterns.md) — the recognisable
  wrong shapes, what each one costs, and the migration out of it.
