# Declaring ObjectTypes — `catalog.refine_object_type`

After a table exists, auto-CRUD has already registered a default
ObjectType from the column shape — every column becomes a `mapped`
`string` property, no links, no description. To declare a **rich**
domain shape — enum-narrowed status columns, typed FK links with
cascade semantics, human-readable descriptions — invoke
`catalog.refine_object_type`. The refinement layers on top of the
default; hand-declared still wins. Successive partial refinements
accumulate — a property/link omitted from a later refinement is
retained from the prior one, not dropped.

See [Phase 17.5 §7](../../../phases/17.5-declarative-handlers.md#7-objecttype-refinement--catalogrefine_object_type)
for the spec and [Phase 17.5 §6](../../../phases/17.5-declarative-handlers.md#6-cascade-on-link--extending-objecttype)
for the cascade-on-link contract.

---

## What you can refine

| Field | Effect |
|---|---|
| `description` | Human-readable summary surfaced in `canon objects list-types` and `schema_describe`. |
| `properties.<name>` | Narrow a `mapped` property's type (e.g., `string` → `enum`), or surface a column that the default registration hid. |
| `links.<name>` | Add (or overwrite) a typed link to another ObjectType: `target`, `cardinality`, `via`, the optional `on_delete` policy, and the optional `part_of` compound marker. |
| `rollups` | Declared root summary properties — each computes an aggregate over a `part_of` child link (compound objects, ADR 0002). |
| `parts_frozen_when` | Lifecycle freeze gate — block all part writes when the root row's state is in a declared set (compound objects). |
| `defaults` | Per-verb opt-out for auto-CRUD: `{ delete: false }` makes the row undeleteable on every surface — the Action ceases to exist. |

Multiple refinements compose; later calls overlay later. The
`_object_type_refinements` table persists each refinement so reboots
re-apply them.

---

## Enum narrowing — the load-bearing pattern

A `lead.status` column starts life as a permissive `string`. To bind a
state machine to it (Phase 18.5) the property must be an `enum` with a
declared finite value set:

```yaml
action: catalog.refine_object_type
input:
  name: lead
  description: Pre-conversion sales prospect — Lead.status FSM-governed.
  properties:
    status:
      kind: mapped
      column: status
      type: enum
      values: [new, contacted, qualified, converted, disqualified]
  links:
    converted_to: { target: customer, cardinality: one, via: id }
```

The action validates the enum **against persisted data** before
applying — if any existing row holds a value outside the proposed set
(say a stale `pending` in the `status` column), the refinement fails
with `VALIDATION` and the offending value in the message. Clean the
data first, then retry.

JSON form for `actions invoke catalog.refine_object_type --input -`:

```json
{
  "name": "lead",
  "description": "Pre-conversion sales prospect — Lead.status FSM-governed.",
  "properties": {
    "status": {
      "kind": "mapped",
      "column": "status",
      "type": "enum",
      "values": ["new", "contacted", "qualified", "converted", "disqualified"]
    }
  },
  "links": {
    "converted_to": { "target": "customer", "cardinality": "one", "via": "id" }
  }
}
```

---

## Enum value colors — `valueColors`

An `enum` property may carry an optional `valueColors` map that assigns a
**semantic color token** to each value. The shape is
`Record<enumValue, token>` — every key must be one of the property's declared
`values`, and every value is one of a **closed set of five white-label tokens**:

| Token | Semantics |
|---|---|
| `hot` | Urgent / destructive — the most-urgent end of the ramp. |
| `warm` | Caution / needs attention. |
| `cool` | Positive / healthy / done. |
| `info` | Informational / in-progress. |
| `neutral` | Default / muted (the fallback for an uncolored value). |

The tokens are **named, not raw hex, on purpose**: they stay white-label, so the
app's brand accent flows through them and the palette can never clash with the
chrome. A value with no entry keeps the default (hashed / convention) chip color.

```yaml
action: catalog.refine_object_type
input:
  name: lead
  properties:
    status:
      kind: mapped
      column: status
      type: enum
      values: [new, contacted, qualified, converted, disqualified]
      valueColors:
        new: info                # informational
        qualified: warm          # needs attention
        converted: cool          # positive / done
        disqualified: neutral    # muted
```

A key that isn't in `values`, or a token outside the five, fails
`catalog.apply` with `invalid_enum_color` — a color that names a value the
column can't hold, or a token the chip can't render, is a defect, not a style
choice. The tokens surface as **status chips** and the **queue card's urgency
border tint**; see the `canonify-viewspecs` skill (`canon skills install
canonify-viewspecs`) for where each renders.

---

## Enum value labels — `valueLabels`

`valueColors`' **sibling**: an optional `Record<enumValue, string>` map that
overrides the **text** each enum value renders as, where `valueColors` overrides
the color. Declared at the same site — on the `enum` property — it turns a raw
stored token into a human label: `health_status` `green` → `"Healthy"`,
`yellow` → `"At risk"`.

```yaml
action: catalog.refine_object_type
input:
  name: account
  properties:
    health_status:
      kind: mapped
      column: health_status
      type: enum
      values: [green, yellow, red]
      valueColors:                 # the color sibling
        green: cool
        yellow: warm
        red: hot
      valueLabels:                 # the text sibling
        green: Healthy
        yellow: At risk
        red: Critical
```

Three properties make it safe to add or change freely:

- **Display-only.** The label is a render-time substitution — the **stored value
  is never altered**. Where a value has no entry (or the enum has no
  `valueLabels` at all), the value falls back to the humanised raw token
  (`green` → `Green`), exactly as before.
- **Never touches query semantics.** Filtering and sorting still key off the
  **raw** stored value — `?where[health_status]=green` matches the row whose
  label reads "Healthy". So you can relabel `green` → "Healthy" → "Good standing"
  without breaking a single saved filter, preset, or board grouping.
- **Renders everywhere the value does** — list column, detail field, status
  chip, board card, queue fact — because it rides the same discovery wire the
  value does.

A key that isn't in `values`, or a **blank/whitespace-only** label, fails
`catalog.apply` with `invalid_enum_label` — a label that names a value the column
can't hold, or one that would render an empty cell, is a defect, not a style
choice (mirroring `invalid_enum_color`). Labels are otherwise free text — there
is no closed palette the way colors have one. Like `values` / `valueColors`,
`valueLabels` is **inherited when omitted** on a re-refine and **shed** when the
property is refined away from `enum`.

> Declaring `valueColors` **without** `valueLabels` earns the advisory
> `elegance_missing_value_labels` — a colored chip that still shows the raw token
> (`green`) reads worse than one that shows "Healthy". See
> [error-codes › *`elegance_*`*](error-codes.md).

---

## Property currency — `currency`

A `number` property may declare the **ISO-4217 currency its amount is
denominated in** — three uppercase letters (`DKK`, `EUR`, `USD`):

```yaml
action: catalog.refine_object_type
input:
  name: invoice
  properties:
    total:
      kind: mapped
      column: total
      type: number
      currency: DKK
```

The rule is **currency travels with the data, locale with the viewer** — there
is **no default currency**. The declared `currency` is the *denomination* — a
fact about the number that rides on the discovery wire — and is distinct from a
ViewSpec `format: "currency"` hint, which is a *viewer-locale rendering*
instruction. The amount is denominated in DKK for everyone; the viewer's locale
only decides symbol placement and digit grouping.

Like `values` / `valueColors`, `currency` is **inherited when omitted** on a
re-refine (a `number` property re-stated without it keeps the prior code) and
**shed** when the property is refined away from `number`. A code that isn't three
uppercase letters — or a `currency` on a non-`number` property — fails
`catalog.apply` with `invalid_currency`. (The check is shape-only, `/^[A-Z]{3}$/`
— it accepts any well-formed code and rejects a symbol like `kr`; `Intl` supplies
the actual symbol and precision at render time.)

---

## Links — what they enable

Links are typed pointers between ObjectTypes; once declared, the
runtime can navigate them:

```sh
canon objects navigate customer cus_… contacts
canon objects navigate customer cus_… deals
```

Link fields:

| Field | Required | Meaning |
|---|---|---|
| `target` | yes | Name of the destination ObjectType. |
| `cardinality` | yes | `one` or `many`. |
| `via` | yes | Column name on the **target** ObjectType (for `many` links) or on the **source** (for `one` links). Example: `contacts` link from customer → contact uses `via: customer_id` (the FK on contact). |
| `on_delete` | no | Referential-action policy: `restrict` (default), `cascade`, `set_null`. Honoured by every `delete` path — auto-CRUD's `<obj>.delete`, declared handler `delete` steps, and direct `runtime.delete()`. |

Reverse-nav example (customer → contacts → deals → notes → leads):

```yaml
action: catalog.refine_object_type
input:
  name: customer
  description: Customer the org sells to — the CRM root entity.
  links:
    contacts: { target: contact, cardinality: many, via: customer_id }
    deals:    { target: deal,    cardinality: many, via: customer_id }
    notes:    { target: note,    cardinality: many, via: customer_id }
    leads:    { target: lead,    cardinality: many, via: customer_id }
```

---

## Cascade-on-link

The `on_delete` field controls what `typedDelete` does when child rows
exist:

| Policy | Behaviour |
|---|---|
| `restrict` (default) | If any child rows exist, refuse with `INVALID_STATE` (details include the link name and child count). |
| `cascade` | Delete child rows first, in dependency order. Atomic — happens in the same tx as the parent delete. |
| `set_null` | Update child rows setting the FK column to `NULL`. Only valid when the FK column is nullable. |

Cascade is **data semantics**, not handler semantics — declare it once
on the link, every `delete` path respects it. See
[Phase 17.5 §6](../../../phases/17.5-declarative-handlers.md#6-cascade-on-link--extending-objecttype).

---

## Compound objects — `part_of`, `rollups`, `parts_frozen_when`

A **compound object** is a root whose identity includes its child parts —
e.g. an invoice and its lines. Three refinement fields opt an ObjectType
into the compound machinery (ADR 0002). They're summarised here; the
`canonify-objects-actions` skill (`canon skills install
canonify-objects-actions`) covers compound reads, writes, and the full
validation contract in depth.

- **`part_of: true`** on a link marks the target as a structural part of
  the source. Only valid on target-side FK links (the `many` side);
  incompatible with `on_delete: set_null`; the `part_of` graph must form
  a forest (single-parent, acyclic).

  ```yaml
  links:
    lines: { target: invoice_line, cardinality: many, via: invoice_id, part_of: true }
  ```

- **`rollups`** declares root summary properties — each a `<agg>` over one
  `part_of` child link, computed at read time. A rollup spec is
  `{ link, agg, column?, type? }`: `agg` is `sum|count|avg|min|max`;
  `column` (the numeric child property) is required for everything except
  `count`; `type` defaults to `number`.

  ```yaml
  rollups:
    total:      { link: lines, agg: sum, column: amount }
    line_count: { link: lines, agg: count }
  ```

- **`parts_frozen_when`** is a lifecycle gate `{ column, in }` — when the
  root row's `column` (a `mapped` property) holds a value in the `in`
  set, all part writes (insert/update/delete) are refused with `CONFLICT`
  / `details.reason: 'FROZEN'`. Creating a born-frozen root still
  succeeds; the gate blocks subsequent mutation.

  ```yaml
  parts_frozen_when: { column: status, in: [locked, paid] }
  ```

All three are validated at `catalog.apply` against the live ObjectType
graph (unknown link, non-`part_of` link in a rollup, non-numeric rollup
column, non-mapped freeze column, empty `in` set all fail `VALIDATION`).
A compound root reads back with its parts nested via
`canon objects get <type> <id> --expand`.

---

## Surfacing hidden columns

Slice-1 platform ObjectTypes (e.g., `customer`) hide physical columns
like `org_id` from the typed shape. If your declared handler needs to
write through one, refine it explicitly:

```yaml
action: catalog.refine_object_type
input:
  name: customer
  properties:
    org_id: { kind: mapped, column: org_id, type: string }
```

The validator checks that the column actually exists on the underlying
table (`PRAGMA table_info`); typos fail at declaration time with
`VALIDATION/<col> does not exist on table "<name>"`.

This is the precise reason
[`tests/scenarios/minimal-crm/session.yaml`](../../../../tests/scenarios/minimal-crm/session.yaml)
refines the customer ObjectType **before** declaring
`lead.convert` — without it, the `insert customer values: { org_id: $principal.org_id }`
step would fail the static type-check at `actions.declare` time
(no `org_id` property on the typed shape).

---

## Opting out of default verbs

`defaults` is the per-verb kill switch:

```yaml
action: catalog.refine_object_type
input:
  name: invoice
  defaults:
    delete: false   # invoice.delete no longer exists on any surface
    update: false   # invoice.update no longer exists either
```

Disabling a verb unregisters the corresponding default ActionType.
That makes room for a custom Action under the same name — declare
`invoice.delete` via [`actions.declare`](declaring-actions.md) with the
exact business semantics you want (e.g., archive instead of physical
delete).

---

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| ObjectType isn't registered | `NOT_FOUND` | Declare the table first via [`schema.propose_migration`](declaring-tables.md). |
| Enum value set doesn't cover persisted data | `VALIDATION` | Add the missing values OR update the offending rows, then retry. |
| Nullable → non-nullable tightening with NULL rows | `VALIDATION` | Backfill the NULL rows first. |
| Link `via` column doesn't exist on the target table | `VALIDATION` | Add the column with `schema.propose_migration` (`ALTER TABLE`) or fix the typo. |
| Link `target` isn't a registered ObjectType | `VALIDATION` | Declare the target first. |

---

## See also

- [Declaring tables](declaring-tables.md) — the prerequisite step
- [Declaring StateMachines](declaring-state-machines.md) — binds to an enum property declared here
- [Declaring custom ActionTypes](declaring-actions.md) — handler steps reference ObjectType properties; static type-check uses this declaration
- [Phase 17.5 §6](../../../phases/17.5-declarative-handlers.md#6-cascade-on-link--extending-objecttype) — cascade-on-link semantics
- [Phase 17.5 §7](../../../phases/17.5-declarative-handlers.md#7-objecttype-refinement--catalogrefine_object_type) — refinement contract
