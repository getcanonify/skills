# Anti-patterns — the shapes to recognise

Each entry: what it looks like, what it costs, the audit code that fires, and
the way out. Every one of these was found in a real, running, valid Canon.

---

## 1. The shadow entity

**Looks like** a table with `customer_name`, `customer_address`, `customer_cvr`,
`contact_person`, `contact_email` — in a Canon that already has `customer`,
`organization` and `contact` ObjectTypes.

**Costs.** Identity becomes a free-text string; four spellings of one company.
Nothing propagates on change. No navigation, no aggregation, no record page.
Related objects become disconnected islands.

**Code.** `C1-shadow-entity` (error).

**Way out.** Add the FK column and the link → backfill by matching the text
(this is the painful step, and it gets worse every day you postpone it) → add
the `kind: derived` property → repoint `display.title` and every view column at
the derived property → drop the text columns **last**. Do not stop before the
drop: a shadow column kept "for safety" diverges within a week and then nobody
knows which one is authoritative.

**The tell during authoring.** You are about to write a column whose name is
`<other_object>_<something>`. That is almost always a link.

---

## 2. The unanchored child

**Looks like** a table with `deal_id`, `customer_id`, `order_id` columns and an
ObjectType whose `links` map contains only `cardinality: many` entries — or is
empty.

**Costs.** No upward navigation, no breadcrumb. A record page cannot scope a
child table to the record it is showing, so the table either shows the whole org
or cannot be placed. `on_delete` has nothing to attach to, so parents delete
into orphans. Dot-traversal column refs in ViewSpecs do not resolve.

**Code.** `C2-unanchored` (error). Its render-side symptom is catalog
validation's `elegance_unlinked_child_table` advisory.

**Way out.** One `catalog.refine_object_type` call per object adding the upward
link. No migration, no backfill — the column already holds the value; you are
declaring what it means.

```yaml
links:
  deal: { target: deal, cardinality: one, via: deal_id, fk_on: source }
```

**The direction trap.** `fk_on: source` (the default is `target`) is what says
"the FK is on *this* object". Omit it on a `cardinality: one` upward link and
the platform looks for `deal_id` on the `deal` table, finds nothing, and you
have declared a link that does not work. If your "anchored" object still cannot
navigate upward, this is why.

---

## 3. The re-implemented primitive

**Looks like** twelve `signature_*` columns; a `document_url` / `document_type`
column pair repeated per attachment; a hand-rolled `approvals` table; a
`change_log` table; a `retry_queue` table; a status column guarded by
hand-written legality checks in every action.

**Costs.** You maintain a worse copy of something tested, audited and supported.
The hand-rolled e-signature in the audited Canon had no consent capture, no
token redaction from the audit trail, no server-derived evidence, no
certificate, and no lifecycle propagation. Twelve columns bought a status
string.

**Code.** `C13-primitive-dup` (error) when three or more columns on one table
match a known primitive's shape.

**Way out.** `canon objects list-types` and `canon actions list --json` in a
**fresh org** — what comes back is the platform surface, everything you get for
free. Then keep one FK to the primitive and derive what you need to display.

**The tell during authoring.** You are about to write your third column with a
shared prefix (`signature_`, `approval_`, `file_`). Search the catalog for that
noun before the fourth.

---

## 4. Float money

**Looks like** `amount REAL`, `price REAL`, `unit_price REAL`, or a `number`
property with no `currency`.

**Costs.** Binary floats cannot represent `0.1`. Totals drift at the minor unit,
and drift grows with the number of lines summed — so the error is largest on the
biggest invoice and at month-end reconciliation, which is exactly where somebody
is checking.

**Code.** `C10-float-money` (error).

**Way out.** `amount_minor INTEGER NOT NULL` plus `currency TEXT NOT NULL`, and
declare the property with its `currency` so it renders as an amount rather than
a count. Migrating existing float rows means multiplying by 100 and rounding —
**decide and document the rounding direction before you run it**, because you
are about to change historical numbers by up to one øre each and you will be
asked about it.

---

## 5. The mutable price

**Looks like** `product.default_price` — one column, updated in place when the
price changes.

**Costs.** History is gone the moment it is overwritten. Last quarter's invoices
cannot be reproduced. A percentage-discount agreement now recomputes against
today's list price. Scheduling a change means someone remembering to run an
update on the right morning. A warehouse has to reconstruct the timeline from
change logs, badly.

**Way out.** A `price_list_item` table with `valid_from` / `valid_to`, and a
`reprice` action that closes the open interval and inserts a new one in one
transaction. Seed it by inserting the current price with `valid_from` set to the
earliest date you can defend. You cannot recover the history you never wrote —
the seed row is the best you get, which is the argument for doing this before
launch rather than after.

See [pricing-worked-example.md](pricing-worked-example.md).

---

## 6. The collapsed zero

**Looks like** resolution code that reads `if (price) { … } else { useDefault() }`,
or a nullable `amount_minor` where `NULL` is used to mean "no agreement".

**Costs.** Two bugs pointing opposite ways. A customer with a negotiated free
line gets billed at list price, or a customer with no agreement at all gets
billed nothing — and neither surfaces until somebody reconciles a quarter.

**Way out.** `amount_minor INTEGER NOT NULL`. Absence is expressed as **no row**,
never as a NULL or zero amount. Resolution asks "did I find a row?", never "is
the amount truthy?". A deliberately free line is inserted like any other price,
with `amount_minor: 0` and its own `valid_from` — and, ideally, a `note` saying
who agreed it.

---

## 7. `$now` as the resolution date

**Looks like** a billing action that resolves prices against `$now` instead of
against the order's `placed_at`.

**Costs.** Silent history rewriting. Re-run last month's billing after a
repricing and every line comes out different, with no error and no trace. This
one is invisible until two runs of the same report disagree.

**Way out.** Resolution takes an explicit `at`, and the caller passes the
transaction's own timestamp. `$now` is correct for stamping *when this happened*
and wrong for asking *what was true then*.

---

## 8. The live report

**Looks like** a saved query, a `metric` block, or a `<period>_summary` view
that recomputes on every open — used as the thing handed to invoicing.

**Costs.** A different answer every week. A late order, a corrected quantity, a
repriced product, a merged customer — any of them silently changes a month that
was already invoiced. What you emailed on 1 July no longer exists anywhere.

**Way out.** A `billing_report` object with stored `billing_report_line` rows, a
`draft → sealed` StateMachine with `sealed` terminal, and no update/delete
action on the lines at all. Corrections become adjustment lines on the **next**
report, carrying a reference to the line they correct.

A live dashboard of the current period is fine and useful — it is just not the
same artefact as the sealed report, and it must not be the one you hand over.

---

## 9. Two models for one concept

**Looks like** `customer` and `client` tables; `deal` and `opportunity`; a
proposal's `line_items` and an order's `order_line` with the same six columns
and different names.

**Costs.** Every query has to know both. Every report picks one and is wrong.
Reconciling them becomes a permanent tax nobody has time to pay off, and it
grows: each new feature has to be built twice.

**Way out.** Decide which is canonical, migrate the other into it, and delete
it. If both genuinely exist in the business — a `customer` who is a billing
entity and an `organization` that is a legal entity — then model **both**, with
a link between them and one clear sentence in each `description` saying what
distinguishes them. That split is correct when it is deliberate and documented;
it is a defect when it is an accident nobody can explain.

---

## 10. The rules engine for two cases

**Looks like** a `priority` integer column, a `scope_rank` lookup table, or a
`pricing_rule` object with a `condition` expression — built to choose between a
list price and a customer override.

**Costs.** You now maintain an interpreter. Its behaviour cannot be read off the
data, it needs its own tests, and its failure mode is a wrong number with no
explanation. The fifth scope it was built for never arrives.

**Way out.** Write the ordered lookup as prose in the ObjectType description and
as an ordered sequence of reads in the action. Two scopes is two cases; three is
three. Reach for generality when you have the fourth, and not before.
