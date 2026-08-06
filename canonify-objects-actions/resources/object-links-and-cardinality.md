# Object links & cardinality

A **link** is a typed edge from one ObjectType to another. Links power
`canon objects navigate <type> <id> <link>` and carry the referential policy
that governs what happens to children when a parent is deleted. This page is
verified against `ObjectLink` / `LinkCardinality` / `LinkOnDelete` in
`packages/canon/src/types.ts` and the `ObjectLinkSpec` input schema in
`packages/canon/src/actions/catalog/refine-object-type.ts`.

## The link shape

Each entry in an ObjectType's `links` map has:

| field | required | meaning |
|---|---|---|
| `target` | yes | name of the ObjectType this link points at |
| `via` | yes | the FK column **on the target table** that points back here (on the **source** table when `fk_on: "source"`) |
| `cardinality` | yes | `one` or `many` |
| `on_delete` | no | referential policy; defaults to `restrict` |
| `fk_on` | no | which table holds the `via` FK column: `target` (default — parent→child) or `source` (a belongs-to / child→parent FK) |
| `part_of` | no | `true` marks the target as a constituent **part** of this root's identity (a compound object — ADR 0002) |

The critical, easy-to-get-wrong field is **`via`**: it names the column on the
*target* table, not on the source. A `customer` with many `task` rows declares
`via: customer_id` because `customer_id` lives on the `task` table.

## Full input + envelope

Refine an ObjectType to add a one-to-many link plus its inverse one-to-one:

```json
{
  "name": "customer",
  "links": {
    "tasks": { "target": "task", "cardinality": "many", "via": "customer_id", "on_delete": "cascade" },
    "orders": { "target": "order", "cardinality": "many", "via": "customer_id", "on_delete": "restrict" }
  }
}
```

```json
{
  "data": { "name": "customer", "refined": true, "refinement_id": "re_…" },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

The validator (`validateRefinement`) checks two things per link:

1. The `target` ObjectType is registered — else
   `"Link "tasks": target ObjectType "task" is not registered"`.
2. The `via` column exists on the target table — else
   `"Link "tasks": via column "customer_id" not found on target table
   "task""`.

So declare the target table (and refine its ObjectType) **before** the link
that points at it.

## Cardinality

- `one` — navigating returns a single row (or null). Use for a child's
  back-reference to its parent (`task.customer`).
- `many` — navigating returns a list. Use for a parent's collection of
  children (`customer.tasks`).

Cardinality is a navigation/render hint; it does not itself enforce a unique
constraint (that lives in the table DDL).

## on_delete — the referential policy

`LinkOnDelete` is `restrict | cascade | set_null`. It is **data semantics, not
handler semantics** — honoured by *every* path that deletes a row: auto-CRUD's
`<obj>.delete`, a custom `actions.declare`-d delete step, and a manual
`objects.delete`. You declare it once on the link.

| policy | behaviour on parent delete |
|---|---|
| `restrict` (default) | refuse with `INVALID_STATE` if any child rows exist; the message cites the link name + child count |
| `cascade` | delete child rows first (depth-first, same transaction); nested cascades chain through their own `on_delete` |
| `set_null` | set the `via` FK column to NULL on every child row |

`set_null` requires the `via` column to be **nullable** on the target
ObjectType — registration rejects `set_null` against a NOT NULL column. If you
need it, refine the target's property to `nullable: true` first (see
[object-property-types.md](object-property-types.md)).

## Choosing a policy

- A child that is meaningless without its parent → `cascade`
  (e.g. line items under an order).
- A child that records history you must not lose → `restrict`
  (e.g. orders under a customer — block the delete, force an archive flow).
- A child that survives an unlinked parent → `set_null`
  (e.g. tasks whose assignee was removed).

## `part_of` — constituent parts of a compound (ADR 0002)

Setting `part_of: true` on a link marks the target as a **structural part of the
root's identity** — `invoice_line` is *part of* `invoice`, not merely linked to
it. This is deliberately distinct from `on_delete`: `cascade` answers "what
happens when the parent is deleted," while `part_of` answers "is this child part
of the parent's identity." A `part_of` link unlocks compound reads
(`objects get --expand`), declared `rollups`, the `<root>.replace` action, and
the `parts_frozen_when` lifecycle gate (see
[actions-handler-patterns.md](actions-handler-patterns.md) and `SKILL.md` Part 1).

```json
{
  "name": "invoice",
  "links": {
    "lines": { "target": "invoice_line", "cardinality": "many", "via": "invoice_id", "on_delete": "cascade", "part_of": true }
  }
}
```

`part_of` carries three constraints, validated at `catalog.apply` (the diagnostic
codes are stable and load-bearing) and re-checked at registry registration:

| constraint | violated by | diagnostic code |
|---|---|---|
| target-side FK only | `part_of` on a belongs-to link (`fk_on: "source"`) — the FK inverts ownership | `part_of_shape_invalid` |
| no orphaning | `part_of` + `on_delete: "set_null"` — a part can't be orphaned from the identity it belongs to (use `cascade` or `restrict`) | `part_of_shape_invalid` |
| NOT-NULL FK | the `via` FK column on the child is declared `nullable: true` — a NULL FK leaves the part ungoverned (no version-anchor bump, no freeze gate) | `part_of_shape_invalid` |
| target must exist | `part_of` targets an ObjectType not declared in this catalog | `part_of_unknown_target` |
| forest shape | a child with **more than one** incoming `part_of` edge (single-parent), or any `part_of` **cycle** including a self-loop `A → A` (acyclic) | `part_of_forest_violated` |

The forest constraint is the key structural rule: the whole `part_of` graph must
be a **forest of trees** — each child has at most one parent, and there are no
cycles. Parts of parts are fine (a 3-level tree); two parents for one child, or
any loop, is not.

## See also

- [object-property-types.md](object-property-types.md) — making a FK column nullable for `set_null`.
- Base skill: `canon skill --name canonify resource declaring-objecttypes.md`.
