# Forms and actions

How to wire ActionTypes into a view: the `panel` form, the `button` trigger,
preset params from the render context, and `showWhen` conditioning. Then the
`view_specs.declare` action that applies a finished view to an org.

---

## Subject auto-bind — don't hand-wire the record id

Most actions operate on a specific object and carry its id as an input — the
action's **subject**. The platform derives the subject from the catalog
(conventionally the input named `` `${object_type}_id` `` — `deal.advance` →
`deal_id`; an ActionType may also declare it explicitly) and surfaces it in
discovery, so every surface knows which input is the operated-on object.

**The payoff for you:** when you drop a `button` or `panel`/`actionForm` into a
**record-scoped view** — a `detail` view, or a composite/button rendered under a
record where `$context.id` exists — the platform **auto-binds the subject to
that record**. You do **NOT** write `presetParams: { deal_id: "$context.id" }`;
the renderer fills it for you. (The auto-default surfaces — list row actions,
the record detail toolbar, board cards — preset the subject from the current
record the same way.)

Only set `presetParams` for the subject param to **override** the auto-bind —
i.e. deliberately target a *different* object than the one in context.
`presetParams` is otherwise for **non-subject** inputs (a literal like a tier,
or another `$context.*` value the form needs). Precedence: an explicit author
`presetParams` entry **>** platform auto-bind from context **>** unset (the form
asks).

In a non-record view (a top-level list or dashboard, no `$context.id`) there is
nothing to bind, so a no-subject action like `<type>.create` correctly asks for
its inputs.

---

## `panel` — a generated action form

A panel hosts a form built from an Action's input schema. The renderer reads
the Action's typed input and lays out the fields; submission goes through the
governed `actions.invoke` path, so policy still applies.

```json
{
  "type": "panel",
  "actionForm": {
    "action": "customer.create",
    "submitLabel": "Add customer",
    "presetParams": { "tier": "standard" }
  }
}
```

```
+--------------------------------------+
|  New customer                        |
+--------------------------------------+
|  Name    [____________________]      |
|  Email   [____________________]      |
|  Tier    [ standard          ▾]      |   <- preset
|                       [ Add customer ]|
+--------------------------------------+
```

- `action` (required) — the ActionType name. Validated against the live
  registry; an unknown action fails with `unknown_action`.
- `submitLabel` (optional) — overrides the default submit caption.
- `presetParams` (optional) — pre-fill input fields. Values may be literals
  or `$context.*` refs resolved from the render context. You do **not** need to
  preset the action's subject (the operated-on record) in a record-scoped view —
  that auto-binds (see "Subject auto-bind" above); use `presetParams` for
  non-subject inputs (here, `tier`) or to override the subject.

---

## `button` — a one-tap trigger

For an Action you can fire without a full form (or with `presetParams`
supplying everything).

```json
{
  "type": "button",
  "action": "deal.advance",
  "label": "Advance stage"
}
```

```
+------------------+
|  Advance stage   |
+------------------+
```

No `presetParams` here: this button lives in a record-scoped view, so the
platform auto-binds the subject (`deal_id`) to the record in scope — the same
button in a master-detail layout advances *the selected deal*. Add a
`presetParams` entry for `deal_id` only to target a **different** deal than the
one in context (see "Subject auto-bind" above).

### Per-action presentation comes from the ObjectType

A button or panel doesn't carry its own emphasis/hidden styling — that's
declared once at the ObjectType-refinement level via `display.actions`, keyed by
the fully-qualified action name. An entry like
`display: { actions: { deal.advance: { emphasis: primary } } }` marks
`deal.advance` as the prominent CTA wherever it renders (auto-default toolbars,
list-row menus, board cards, and the `button` blocks above); `{ hidden: true }`
drops it from those action surfaces (a **display** gate only — the action stays
invokable and governed), and `{ label: "…" }` overrides its caption. So prefer
declaring presentation on the ObjectType over hand-tuning each `button` — the
authoritative reference is `canonify-app-authoring` ›
*Refine the ObjectType* › **Per-action presentation — `display.actions`**.

---

## `showWhen` — conditional display

Every block accepts `showWhen`: an array of predicates using the same grammar
forms use — `{ field, op, value? }`, `op` ∈ `eq` / `in` / `neq` / `present`.
The block renders only when **all** predicates hold.

```json
{
  "type": "panel",
  "actionForm": { "action": "order.refund", "submitLabel": "Issue refund" },
  "showWhen": [{ "field": "status", "op": "eq", "value": "paid" }]
}
```

```
status = "paid"   ->  +----------------------+
                      |  Issue refund   [ ▸ ] |   panel shown
                      +----------------------+

status = "draft"  ->  (panel not rendered)
```

> **UX, not security.** `showWhen` hides UI; it does **not** stop the data
> being fetched, and it runs client-side. Never rely on it to withhold
> sensitive data — that is server-side policy + column projection.

---

## A form + action composite

A realistic edit screen: a detail of the record, a panel to edit it, and a
button for a state transition.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "detail",
      "source": { "objectType": "deal" },
      "properties": ["name", "amount", "stage"]
    },
    {
      "type": "panel",
      "actionForm": {
        "action": "deal.update",
        "submitLabel": "Save"
      }
    },
    {
      "type": "button",
      "action": "deal.advance",
      "label": "Advance stage",
      "showWhen": [{ "field": "stage", "op": "neq", "value": "won" }]
    }
  ]
}
```

The panel and button sit under a record (`$context.id` is in scope), so both
auto-bind their subject (`deal_id`) to the deal — no `presetParams` wiring.

---

## Applying a view: `view_specs.declare`

A ViewSpec is just a document until you declare it. The `view_specs.declare`
ActionType attaches it to an org. Its input is `{ name, object_type, spec }`:

```json
{
  "name": "Deal.detail",
  "object_type": "deal",
  "spec": {
    "type": "detail",
    "source": { "objectType": "deal" },
    "properties": ["name", "amount", "stage"]
  }
}
```

- `name` — a dot-separated handle (e.g. `Deal.detail`). Re-declaring the same
  name overrides the previous spec (idempotent per-key).
- `object_type` — must equal the spec's `source.objectType` for a **top-level**
  `table` / `detail` / `board` block; a mismatch fails with
  `source_object_type_mismatch`. Two exceptions: `metric` sources may target any
  ObjectType the org owns, and a **composite child** may bind a *different*
  ObjectType than the top-level one (a compound view — e.g. an `invoice` detail
  over an `invoice_line` table; see the master-detail resource). A child only
  fails `source_object_type_mismatch` when its `source.objectType` names an
  ObjectType that doesn't exist in the registry.
- `spec` — the ViewSpec itself.

The action validates against the live registry before persisting. Failure
codes: `unknown_object_type`, `source_object_type_mismatch`,
`unknown_column_ref`, `unknown_action`.

Invoke it isomorphically (the composition-patterns resource shows all three
surfaces side by side):

```sh
canon actions invoke view_specs.declare --input @deal-detail.json
```
