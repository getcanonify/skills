---
title: Chapter 2 — Refine the ObjectType
description: Use catalog.refine_object_type to upgrade the permissive auto-default into a rich domain shape — narrow string columns to enums (the prerequisite for a StateMachine), add typed links with on_delete policies, surface hidden columns, and write human descriptions.
---

# Chapter 2 — Refine the ObjectType (`catalog.refine_object_type`)

Chapter 1 left you with a working but **permissive** ObjectType: every
column is a `mapped` `string`, there are no links, and there is no
description. `catalog.refine_object_type` upgrades it into the rich domain
shape — the form the platform (and the renderer in Chapter 5) actually
reasons about.

> The `customer` / `contact` / `deal` shapes below are illustrative.

## The action

| | |
|---|---|
| **Action** | `catalog.refine_object_type` |
| **Input** | `{ name: string, description?: string, properties?: {...}, links?: {...}, defaults?: {...}, display?: { title?, subtitle?, formFields?, primaryProperties?, actions? }, browsable?: boolean, rollups?: {...}, parts_frozen_when?: {...} }` |
| **Envelope** | `{ data: { name, refined: true }, audit_event_id, approval_status: 'not_required' }` |

Refinement is **additive and idempotent** in intent — you describe the
shape you want, and the runtime validates it against the live table and
persisted data before persisting the refinement.

## Enum narrowing — the load-bearing pattern

A StateMachine (Chapter 3) can only bind to a typed `enum` property. So the
first job is to narrow each state column from a free `string` to an `enum`
with an explicit value set:

```yaml
action: catalog.refine_object_type
input:
  name: deal
  description: A sales opportunity against a customer — Deal.stage FSM-governed.
  properties:
    stage:
      kind: mapped
      column: stage
      type: enum
      values: [open, proposal_sent, won, lost]
```

The validator checks the enum **against persisted data** — if a row
already holds a `stage` value not in your set, the refinement fails with
`VALIDATION` and the offending value in the message. Clean the data or
widen the value set.

### Enum value colors — `valueColors`

An enum property may add an optional `valueColors` map — `Record<enumValue,
token>` — giving each value a semantic color for status chips and the queue's
urgency **border tint**. The token is one of the five white-label tokens
(`hot | warm | cool | info | neutral`); the base `canonify` skill's
[declaring-objecttypes › *Enum value colors*](../../canonify/resources/declaring-objecttypes.md)
carries the full table and semantics.

```yaml
input:
  name: deal
  properties:
    stage:
      kind: mapped
      column: stage
      type: enum
      values: [open, proposal_sent, won, lost]
      valueColors: { won: cool, lost: hot, proposal_sent: warm }
```

**Inherit-when-omitted on re-refine.** `valueColors` follows the same rule as
`values`: a later refinement that re-states the property but omits `valueColors`
**keeps** the prior colors; one that refines the property *away* from `enum`
sheds them. A key not in `values`, or an unknown token, fails `catalog.apply`
with `invalid_enum_color`.

### Enum value labels — `valueLabels`

The **text** sibling of `valueColors`: `Record<enumValue, string>`, declared on
the same enum property, overriding what each value **reads as** (`valueColors`
overrides its color). A raw token becomes a human label — `green` → "Healthy" —
everywhere the value renders (list, detail, chip, board, queue).

```yaml
input:
  name: account
  properties:
    health_status:
      kind: mapped
      column: health_status
      type: enum
      values: [green, yellow, red]
      valueColors: { green: cool, yellow: warm, red: hot }
      valueLabels: { green: Healthy, yellow: "At risk", red: Critical }
```

It's **display-only**: the stored value is untouched, and — importantly —
**filtering and sorting still key off the raw value** (`?where[health_status]=green`
matches the "Healthy" row), so you can relabel freely without breaking a saved
filter, preset, or board grouping. A value with no entry falls back to the
humanised token (`green` → `Green`). Same **inherit-when-omitted** rule as
`valueColors`; a key not in `values`, or a blank label, fails `catalog.apply`
with `invalid_enum_label`. The base `canonify` skill's
[declaring-objecttypes › *Enum value labels*](../../canonify/resources/declaring-objecttypes.md)
carries the full semantics. Declaring `valueColors` **without** `valueLabels`
draws the `elegance_missing_value_labels` advisory.

### Property currency — `currency`

A `number` property may declare a `currency` — the **ISO-4217 code the amount is
denominated in** (three uppercase letters, e.g. `DKK`, `EUR`, `USD`):

```yaml
input:
  name: deal
  properties:
    amount:
      kind: mapped
      column: amount
      type: number
      currency: DKK
```

The principle is **currency travels with the data, locale with the viewer** —
there is **no default currency**, and the denomination is a fact about the
number, not a rendering choice. Keep this distinct from a ViewSpec
`format: "currency"` hint: `format` is a **viewer-locale rendering** instruction
(how to *show* a number — symbol placement, grouping), while the property's
declared `currency` is the **denomination** that travels with the value on the
discovery wire. A DKK amount stays DKK regardless of who views it.

`currency` follows the same **inherit-when-omitted** rule as `values` /
`valueColors`: a `number` property re-stated without `currency` keeps the prior
one, and a refinement that changes the type *away* from `number` sheds it. A code
that isn't three uppercase letters, or a `currency` on a non-`number` property,
fails `catalog.apply` with `invalid_currency`.

## Typed links

Links give the platform typed navigation between ObjectTypes. Declare the
forward link on the child and the reverse links on the parent:

```yaml
action: catalog.refine_object_type
input:
  name: deal
  links:
    customer: { target: customer, cardinality: one, via: customer_id }
```

```yaml
action: catalog.refine_object_type
input:
  name: customer
  description: The CRM root entity — an organisation we sell to.
  properties:
    status:
      kind: mapped
      column: status
      type: enum
      values: [prospect, active, churned]
  links:
    contacts: { target: contact, cardinality: many, via: customer_id, on_delete: cascade }
    deals:    { target: deal,    cardinality: many, via: customer_id, on_delete: restrict }
```

`via` names the foreign-key column. `cardinality` is `one` or `many`.

### `on_delete` policies

Honoured by **every** delete path, not just declared handlers:

- `restrict` (default) — deleting the parent fails with `INVALID_STATE` if
  children exist.
- `cascade` — children are deleted first, in the same transaction.
- `set_null` — the children's FK column is nulled.

## Compound objects — `part_of`, `rollups`, `parts_frozen_when`

A **compound object** is a parent whose children are *constituent parts* of its
identity (an invoice and its line items, an order and its lines) — not just
related rows. Three optional fields declare one. They're shape-checked at
declare time and **semantically validated at `catalog.apply`** (the codes below
are `error`-level — they block apply, even non-strict).

### `links.<name>.part_of` (boolean)

Mark a child link as a part by setting `part_of: true` on the **forward link
declared from the parent**:

```yaml
input:
  name: invoice
  links:
    lines: { target: invoice_line, cardinality: many, via: invoice_id, on_delete: cascade, part_of: true }
```

Constraints the validator enforces (`part_of_shape_invalid` /
`part_of_forest_violated`):

- A `part_of` link **may not** be a belongs-to link (`fk_on: 'source'`), and
  **may not** carry `on_delete: 'set_null'` — a part can't outlive its root
  with a nulled parent. (Pair `part_of` with `cascade`.)
- The part_of graph must be a **forest**: a child may have **at most one**
  `part_of` parent, and there may be **no cycles** (including a self-link).

### `rollups` — declared root summary properties

A rollup is a computed property on the **root** that aggregates a `part_of`
child link. Map a result-property name to `{ link, agg, column? }`:

```yaml
input:
  name: invoice
  rollups:
    total: { link: lines, agg: sum, column: amount }
    line_count: { link: lines, agg: count }
```

`agg` is one of `sum | count | avg | min | max`; `column` is required for the
numeric aggregations and omitted for `count`. The link must be a `part_of` link
and the column must exist and be numeric, else apply fails with `rollup_invalid`.

### `parts_frozen_when` — lifecycle freeze gate

Declare a predicate on the root that makes its parts **immutable** while it
holds — e.g. lock an invoice's lines once it's `sent` or `paid`:

```yaml
input:
  name: invoice
  parts_frozen_when: { column: status, in: [sent, paid] }
```

`column` names a **mapped property on the root**, `in` is the non-empty set of
values that freeze the parts. Declaring it on a non-compound type, or naming an
unknown / non-mapped column, fails apply with `parts_frozen_invalid`.

## Display & titles — every object needs a human title

The `display` block tells the renderer (and search, and action forms) how to
show a record to a **human**. It carries `title`, `subtitle`, `formFields`,
`primaryProperties`, and `actions` (per-action presentation, below) — all
optional, all additive. Its keystone is `title`: **every ObjectType must
resolve to a human title, never a raw id.** A `customer` row keyed `cus_01H…`
must headline its company name, not the ULID. There are three strategies:

```yaml
action: catalog.refine_object_type
input:
  name: customer
  display:
    # 1) property — read a column off the row (the common case)
    title: { property: name }
```

- **`{ property }`** — read that column. Use when the row carries a name.
- **`{ template }`** — interpolate `{field}` tokens over the row, for title-less
  rows like audit/event logs: `title: { template: "{from_state} → {to_state}" }`
  renders "onboarding → trial".
- **`{ derived: { via } }`** — borrow the title of the record a FK points to.
  A `customer` has no name of its own, so it derives from its organization:
  `title: { derived: { via: organization_id } }` → headlines the org's name.

> **Declare-time check.** If an ObjectType would fall through to the raw id
> (no name-ish property, no `template`, no `derived`), it has no resolvable
> title — declare one. The Pax canon build fails fast on a title-less object.

### Curating the create/edit form — `formFields`

By default a create/edit form shows every column minus auto-managed ones
(`id`/`created_at`/`updated_at`), with optional fields collapsed under a
disclosure. For a wide ObjectType that dumps too much, declare `display.formFields`
— an ORDERED list of exactly the fields the form should show (reused by the
`<type>.create` and `<type>.update` forms):

```yaml
input:
  name: customer
  display:
    formFields: [organization_id, lifecycle_state, ordering_model]
# the form shows only these (in order); operational/auto columns drop out
```

Omit it and the smart-auto default applies. Declaring it makes a many-column
object's form intentional instead of a dump of every field.

### Disambiguation — when titles collide

Titles aren't unique (two subsidiaries can both be "Novo Foods ApS"). When a
result set collides, the renderer **walks the record's fields for the first one
that distinguishes them, ending at `id`** — automatic, zero-config. Declare a
`subtitle` (same three strategies) to promote the *meaningful* key to the front
of that walk:

```yaml
input:
  name: organization
  display:
    title:    { property: name }
    subtitle: { template: "CVR {cvr}" }   # two "Novo Foods ApS" → distinct CVRs
```

A `customer` can derive its disambiguator from its org too:
`subtitle: { derived: { via: organization_id } }`. The scan order is: declared
`subtitle` field(s) → `primaryProperties` → other non-system fields →
`created_at` → `id` (last resort).

### Per-action presentation — `display.actions`

`display.actions` overrides how *individual actions* on this ObjectType are
presented in a generated UI — which to emphasise, which to relabel, which to
hide from the action surface. It's a record keyed by the **fully-qualified
action name** (`<object_type>.<verb>` — e.g. `customer.create`, `deal.advance`),
each value an override:

```yaml
input:
  name: deal
  display:
    actions:
      deal.advance: { emphasis: primary }              # the prominent CTA
      deal.update:  { label: "Edit deal" }             # relabel the button
      deal.delete:  { hidden: true, emphasis: secondary }
```

Each override carries three **optional** fields (all may be omitted; an absent
key means the action keeps its own defaults):

- **`hidden`** (boolean) — suppress the action from the action surface
  (toolbars, row menus, board cards). **Display-only, not authorization**: a
  hidden action is still invokable programmatically over CLI / REST / MCP and is
  still subject to the same governance gate — `hidden` removes the *button*, not
  the *permission*. To actually withhold an action, use policy + governance, not
  this flag.
- **`label`** (non-empty string) — override the human caption shown on
  buttons/menus (otherwise the renderer humanises the action name).
- **`emphasis`** — `'primary'` (bold CTA) or `'secondary'` (subdued / ghost
  button). Optional; when omitted the prominence is renderer-determined.

**Additive merge across refinements.** Like `links` and `rollups`, the map is
merged key-by-key onto the live ObjectType (`{ ...current, ...incoming }`). A
later refinement that names *new* action keys adds them without clobbering keys
an earlier refinement set but didn't restate — so you can layer per-action
overrides across several `refine_object_type` calls. (Re-stating an existing key
replaces that one entry.)

**Relationship to the action's own `presentation`.** An ActionType can carry its
own [`presentation` block](../../canonify/resources/declaring-actions.md)
(`label`, `group`, `emphasis`, `icon`, `confirm`, `success`, …) declared at
`actions.declare` time. `display.actions` is the **override layer** on top of it,
and the precedence is exact:

- **`label` and `emphasis` here win** over the action's `presentation` — this is
  the per-object relabel / re-emphasise knob.
- **`hidden` here removes the button** entirely (still invokable programmatically;
  display-only, as above).
- **`group` is NOT overridable here** — grouping comes *only* from the action's
  `presentation.group`, its declarative home, so an override never re-buckets an
  action. Likewise `destructive` / `confirm` / `success` pass through from the
  action unchanged. Put grouping and confirm/toast copy on the *action*; use
  `display.actions` for per-object relabels, emphasis, and hiding.

> This is the ObjectType-level home for per-action presentation *overrides*.
> ViewSpec forms and buttons render against the same `emphasis` / `hidden`
> semantics — see `canonify-viewspecs` › *Forms and actions*.

## Browse-data visibility — `browsable`

`browsable` (a top-level boolean on the refinement) controls whether the
ObjectType appears in the web app's **Browse-data** navigation:

```yaml
action: catalog.refine_object_type
input:
  name: signature_request
  browsable: false
```

- `false` **hides** the type from the Browse sidebar **and** suppresses its
  auto-generated `<type>.list` default view — the opt-out for an internal /
  system-support type an app doesn't want operators browsing directly.
- Explicit `true` clears a prior opt-out.
- **Omitted leaves the current value alone** (partial-diff semantics), so a later
  refinement that doesn't restate `browsable` won't accidentally un-hide a type
  you hid.

## Invoke it — three surfaces

**CLI:**

```sh
canon actions invoke catalog.refine_object_type --input @deal-refine.json
```

**REST** (`{ input, idempotency_key? }` body):

```sh
curl -X POST https://api.canonify.app/v1/actions/catalog.refine_object_type \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "name": "deal",
      "description": "A sales opportunity against a customer.",
      "properties": {
        "stage": { "kind": "mapped", "column": "stage", "type": "enum", "values": ["open", "proposal_sent", "won", "lost"] }
      },
      "links": {
        "customer": { "target": "customer", "cardinality": "one", "via": "customer_id" }
      }
    }
  }'
```

**MCP** — the `actions.invoke` tool:

```json
{
  "name": "actions.invoke",
  "arguments": {
    "name": "catalog.refine_object_type",
    "args": {
      "name": "deal",
      "description": "A sales opportunity against a customer.",
      "properties": {
        "stage": { "kind": "mapped", "column": "stage", "type": "enum", "values": ["open", "proposal_sent", "won", "lost"] }
      },
      "links": { "customer": { "target": "customer", "cardinality": "one", "via": "customer_id" } }
    }
  }
}
```

## Verify the refinement

```sh
canon schema describe --object deal              # the rich shape
canon objects list-types                         # confirm deal is there
canon objects navigate customer cust_… deals     # the reverse link works
```

The `navigate` call returns the deal rows for one customer, proving the
typed link resolves.

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| Enum doesn't cover existing rows | `VALIDATION` | widen `values` or fix the offending rows; the message names the value |
| Link `via` column doesn't exist | `VALIDATION` | the FK column must exist on the child table — add it in a Chapter-1 migration |
| Deleting a parent with children | `INVALID_STATE` | the link's `on_delete` is `restrict` — delete children first or set `cascade` |

## Next

→ [Chapter 3 — Bind a StateMachine](state-machines-binding.md): bind a
finite-state machine to the `deal.stage` enum you just narrowed.
