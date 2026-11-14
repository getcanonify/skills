---
name: canonify-domain-modelling
description: The four obligations every Canonify domain model must meet — reuse the canonical object instead of retyping it, anchor every child to its owner, reuse the platform primitive before building one, and model time and money as effective-dated append-only rows with snapshotted settlements. States each as a rule with a worked before/after, and names the retained coherence-audit codes (C1-shadow-entity, C10-float-money) plus their authoring fixes. Read after the base `canonify` skill and before you declare your first table.
version: 2.0.0
---

# Canonify Domain Modelling — the four obligations

The mechanics skills say what you *can* declare. This one says what you *must*.
Every example comes from a real production canon (54 ObjectTypes, 73 actions,
57 tables) that decoded, applied and ran — and broke all four rules. Three of the
four are nearly impossible to retrofit once data piles up, so read this before
your first `schema.propose_migration`.

| # | Obligation | Audit signal |
|---|---|---|
| 1 | **Reuse the canonical object.** Never retype an entity that has an ObjectType. | `C1-shadow-entity` (warning) |
| 2 | **Anchor every child.** Every FK column carries an upward link. | link checks |
| 3 | **Reuse the platform primitive.** Check the catalog before modelling a capability. | catalog discovery |
| 4 | **Effective-date time, snapshot money.** Dated append-only rows; integer minor units. | `C10-float-money` (error) |

`canon catalog audit <dir>` reports the codes; the lifecycle rules for running it
are in [`canonify-app-lifecycle`](../canonify-app-lifecycle/SKILL.md).

## Obligation 1 — Reuse the canonical object

> **Rule.** A property whose value names a real-world entity that already has an
> ObjectType is a defect. Keep an FK, link it to that ObjectType, and let the
> FK render the linked record's title — never a mapped text column.

**Before** — a `proposal` table in a canon that already had `customer`,
`organization` and `contact`:

```sql
CREATE TABLE proposal (
  id               TEXT PRIMARY KEY,
  customer_name    TEXT NOT NULL,   -- display.title is "{customer_name}"
  customer_address TEXT,
  customer_cvr     TEXT,            -- duplicates organization's CVR
  contact_email    TEXT,            -- duplicates contact
  deal_id          TEXT,            -- an FK with no link over it
  ...
)
```

Identity becomes free text ("Nordic Foods A/S", "nordic foods" are two
customers); a customer's move never reaches its proposals; `canon objects
navigate customer cus_… proposals` returns nothing; "proposed value per customer"
is a string match. The audit reports `C1-shadow-entity`, once per column — a
warning, since a name can match by chance: a partner operator's `driver_name`
names no `driver` record, so keep that column and read past the warning.

**After** — keep one FK and declare the link; nothing else is needed to show
the customer's name:

```yaml
action: catalog.refine_object_type
input:
  name: proposal
  links:
    customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
    deal:     { target: deal,     cardinality: one, via: deal_id,     fk_on: source }
  display:
    title: { derived: { via: customer } }
```

An FK with a declared link renders as its linked record's `display.title`
everywhere — table cells, the record header, pickers, search — so a derived
`customer_name` would only shadow it. `display.title: { derived: { via:
customer } }` titles the record through the link — the proposal's identity
*is* the customer's. A linked value that is not its title — the customer's
tier — is a lookup (`kind: lookup`, `link`, `property`: read per reader,
filters, sorts, keeps the target's chip); derive (`kind: derived`, a scalar
SQL expression through the link's FK, never written by a handler) a value you
compute from linked rows. (Title and property strategies:
[object-property-types.md](../canonify-objects-actions/resources/object-property-types.md).)

| The column holds… | Do this |
|---|---|
| A person, company, product, place, account | Link. The FK shows the record's title. |
| Another entity's identifier (CVR, VAT no., a known contact's email) | Link — it is that entity's property. |
| Something true only of this record now (quantity, note, status) | Mapped column. |
| A value fixed at settlement that must never change | Mapped column **plus** a link to its source row (Obligation 4). |

**Migrating out:** add the FK and link → backfill the FK by matching the text (a declared action: `read_many` the rows, `update` each — a migration writes no rows) →
repoint `display.title` and every view column from the text column to the FK →
drop the text column **last**. A shadow column kept "for safety" diverges within a week.
The tell while authoring: a column named `<other_object>_<something>`.

## Obligation 2 — Anchor every child

> **Rule.** An ObjectType with an FK column naming another ObjectType declares
> the upward link. A downward `many` link on the parent is no substitute.

Without the link the platform does not know the column is a reference: no
navigation to the owner, no record page that scopes a child table to its record
(the renderer needs the link to build the filter), nothing for `on_delete` to
attach to (parents delete into orphans), no `customer.organization.name` column
refs, and no settling the child with its owner (a `part_of` / `cascade` child
is locked when its owner reaches a terminal state unless the link declares
`settles: false`, as an activity child such as a comment does — the `canonify-objects-actions`
skill's [state-machines.md](../canonify-objects-actions/resources/state-machines.md#a-terminal-state-settles-the-record)). In the audited canon 24 of 54 types had an unlinked FK; `proposal`
declared five downward links and nothing upward.

```yaml
links:
  # Upward — the anchor. fk_on: source = the FK column is on THIS object.
  deal:     { target: deal,     cardinality: one, via: deal_id,     fk_on: source }
  # Downward — via names the FK column on the TARGET.
  comments: { target: proposal_comment, cardinality: many, via: proposal_id, on_delete: cascade }
```

Omitting `fk_on: source` on an upward link makes the platform look for `deal_id`
on `deal` — the most common way an "anchored" object stays unanchored. Adding
the link needs no migration: the column already holds the value. Check: for
every `*_id` column in `canon schema describe --object <type>`, if the prefix
names a type in `canon objects list-types`, that object owes an upward link.

## Obligation 3 — Reuse the platform primitive

> **Rule.** Before modelling a capability, check whether the platform ships it.
> If it does, compose it. A second implementation is a defect even when it works.

Run `canon objects list-types`, `canon actions list --json` and `canon schema
describe` **in a fresh org** once: what comes back is the platform surface.
Search it for your capability's nouns before writing columns;
`canon actions describe <name> --json` prints anything close.

The audited canon hand-built e-signature: ten ObjectTypes and 12 `signature_*`
columns on `proposal` alone, referencing `signature_envelope` zero times. It
lacked consent capture, `link_token` redaction from the audit trail,
server-derived signing evidence, a verifiable certificate and propagating
terminal lifecycles. It also stored files as url/type column pairs where a
`type: 'file'` property already exists. The fix is one link and one derived
property:

```yaml
links:
  signature_envelope: { target: signature_envelope, cardinality: one, via: signature_envelope_id, fk_on: source }
properties:
  signing_status:
    kind: derived
    type: string
    expression: SELECT status FROM signature_envelope WHERE signature_envelope.id = proposal.signature_envelope_id
```

| You are about to build… | Look for |
|---|---|
| Signing, countersigning | `signature_request`, `signature_envelope`, `signature`, `member_signature` ([esign-actions.md](../canonify-objects-actions/resources/esign-actions.md)) |
| Uploads, attachments | one file per record: a property of `type: 'file'`; any number: the record's attachments, ruled by the type's `attachments: { accept, max_bytes }` and filled by a `multiple: true` file input's handler `attach` ([files](../canonify-isomorphism/resources/runtime-contract.md)) |
| An approval gate | `governance.requires_approval` / `requires_approval_when` |
| An audit / change log | `_audit_event` — every invocation writes one; `canon audit` reads it |
| A retry queue, send-later, webhooks | the outbox (handler or state-machine hooks) |
| A status plus hand-written legality checks | a StateMachine on an enum property |
| A "has this run?" guard table | the action's `idempotency` posture |
| Prices / rates in force over periods | `temporal` on the ObjectType (Obligation 4) |

The tell: your third column with a shared prefix (`signature_`, `approval_`,
`file_`).

## Obligation 4 — Time and money

> **Rule.** A value that changes over time and matters financially is an
> **effective-dated, append-only row**, never a mutable column. Money is
> **integer minor units**, never a float. A settled transaction **snapshots** the
> value it used and **references** the row it came from.

The wrong version works until the first month-end close, and then the history
you needed was never written. It applies to prices, rates, fees, salaries,
commission and tax percentages, SLA thresholds. The whole of it, built end to
end: [resources/pricing-worked-example.md](resources/pricing-worked-example.md).

**4.1 Effective-dated rows.** Never update or delete a price: close the current
period and open a new one. Declare it — `temporal: { from: valid_from, to:
valid_to, key: [product_id] }` on the ObjectType generates `<type>.set` (closes
the open period at the new `from`, inserts, refuses any overlap) and
`<type>.end`, a one-open-period-per-key index, `as_of` reads and the `$as_of`
filter token (contract:
[object-property-types.md](../canonify-objects-actions/resources/object-property-types.md)).
Periods are half-open — `from` inclusive, `to` exclusive, open end `NULL` — so
consecutive periods abut with no date arithmetic. You get reconstructable
history ("what did we charge in February?"), scheduling (a price agreed today
for 1 April is just a row; nothing flips at midnight), and a type-2 dimension a
warehouse consumes as-is. If you hand-model an interval instead, close and insert
in **one** action (one transaction) so a product is never left unpriced.

**4.2 Resolve by specificity, on the transaction's date.** The most specific row
in effect wins: a customer override, else the list price. Write that as an
ordered two-case lookup — not a priority column, scope-ranking table or rules
engine; a third scope is a third case. Resolve against the date the transaction
happened (`order.placed_at`), never `$now`: re-running last month's billing
against today rewrites February silently. `$now` stamps *when this happened*; it
never answers *what was true then*.

**One org-wide rate** (instrument time at 650 DKK/h) is the same rule with a
single series: a table `instrument_rate(id, kind TEXT NOT NULL DEFAULT
'instrument', per_hour INTEGER NOT NULL, valid_from TEXT NOT NULL, valid_to
TEXT, …)` declared `temporal: { from: valid_from, to: valid_to, key: [kind] }`
(`key` needs a column), seeded with one `instrument_rate.set`. The analysis
reads it on its own date, as a derived money property, rounding half-up per
analysis: `(analysis.instrument_minutes * (SELECT per_hour FROM instrument_rate
WHERE valid_from <= analysis.received_at AND (valid_to IS NULL OR valid_to >
analysis.received_at)) + 30) / 60`. A literal `65000` in the expression is
lighter and costs exactly this: a rate change reprices every analysis not yet
snapshotted (4.4), and when the old rate applied is lost.

**4.3 A row with `0` is not the same as no row.** `0` means deliberately free;
no row means no agreement — fall through. Keep the amount `NOT NULL`, express
absence as no row, and ask "did I find a row?", never "is the amount truthy?".
Collapsing them bills a free line at list price, or an unagreed line at nothing.

**4.4 Snapshot onto the transaction line.** At settlement, write the resolved
amount onto the line **and** a reference to the row it resolved from
(`price_source_kind`, `price_source_id`, `priced_at`). The snapshot answers
*what did we charge?*; the dated rows answer *what was agreed on 3 March?*.
Renegotiating in March cannot rewrite February — and ending the line's state
machine in a `terminal` state makes the generated edit refuse it
([state-machines.md](../canonify-objects-actions/resources/state-machines.md#a-terminal-state-settles-the-record)).
This is the documented exception
to Obligation 1: a deliberate, immutable copy that points at its origin.

**4.5 Percentages make list-price history load-bearing.** "12% off list" is a
function of the list price on the transaction date: date the discount row like a
price, resolve both on the same date, and snapshot the computed amount with both
source ids. Update a list price in place and every historical percentage
recomputes. Store an agreed rate (discount, VAT, commission) as integer basis points (`1200` = 12%), not `0.12`,
and declare `scale: { basis: 10000, as: "percent" }` once on it — every view reads 12%, no derived `bp / 10000.0`
twin ([`scale`](../canonify-objects-actions/resources/object-property-types.md#currency-unit-scale)); a ratio with
`format: "percent"` is only for a figure you derive, never store (a share of a limit). Prefer an absolute amount when one was agreed.

**4.6 Money is integer minor units.** `REAL` cannot represent `0.1`; totals
drift, most on the biggest invoice and at reconciliation. Declare `type: money`
with its ISO-4217 `currency` (it stores integers, refuses fractions and renders
as an amount); carry the currency on the row and never mix currencies in one sum
without a dated conversion. The audit reports a `REAL` money column as
`C10-float-money`. Migrating float rows changes history by up to one minor unit
each — decide and document the rounding first.

**4.7 State the rounding policy.** Where (per line, usually — the total is then
an integer sum, never rounded again), to how many places (whole øre), which
direction (half-up or half-even; unstated is the defect). Write it in the
ObjectType description and the action that applies it. Handler rounding:
[actions-handler-patterns.md](../canonify-objects-actions/resources/actions-handler-patterns.md).

**4.8 Periodic reports are sealed objects.** What you handed to invoicing for
June is a stored `billing_report` with stored lines and a `draft → sealed` state
machine (`sealed` terminal, no update/delete on lines) — not a query that
recomputes. Corrections land **forward**, as an adjustment line on July's report
referencing the June line. The same holds for a sent invoice, a filed VAT
return, a signed proposal. A live dashboard of the current period is fine; it is
just not the artefact you hand over.

## Two more shapes to recognise

| Shape | Way out |
|---|---|
| **Two models for one concept** — `customer` and `client`, `deal` and `opportunity`, two line tables with the same columns | Pick the canonical one, migrate, delete the other. If both truly exist (billing customer vs legal organization), model both with a link and one sentence in each `description` saying what distinguishes them. |
| **A rules engine for two cases** — a `priority` column or `pricing_rule` interpreter choosing between list and override | An ordered sequence of reads in the action, described in prose on the ObjectType. Generalize at the fourth case, not before. |

## The checklist

Before every migration, and again before its refinement:

- [ ] No column names an entity that has an ObjectType (`C1-shadow-entity`).
- [ ] Every `*_id` column has an upward link with `fk_on: source`.
- [ ] I ran `canon objects list-types` and `canon actions list --json` before modelling this capability.
- [ ] Every money column is `INTEGER` minor units with a currency (`C10-float-money`).
- [ ] Every value that changes over time is effective-dated and written append-only.
- [ ] Every settled line carries the snapshot **and** the id of the row it came from.
- [ ] The rounding policy is written down.
- [ ] `0` and *no row* are distinguishable everywhere they are read.
- [ ] Every periodic report is a sealable stored object.
