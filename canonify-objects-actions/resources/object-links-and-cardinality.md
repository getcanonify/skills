# Links, cardinality, compound objects

A **link** is a typed edge from one ObjectType to another, declared in
`catalog.refine_object_type`'s `links` map. Links power navigation
(`canon objects navigate customer cus_… contacts`), linked sections on record
pages, link aggregates in views, and the delete policy for children.

## The link shape

| Field | Required | Meaning |
|---|---|---|
| `target` | yes | the ObjectType the link points at (must be registered) |
| `cardinality` | yes | `one` (navigates to one row or null) or `many` (a list). A render/navigation hint, not a uniqueness constraint — that lives in the DDL. |
| `via` | yes | the FK column — on the **target** table by default, on the **source** when `fk_on: "source"` |
| `fk_on` | no | `target` (default: parent → children) or `source` (a belongs-to: the FK is on this row) |
| `on_delete` | no | `restrict` (default) \| `cascade` \| `set_null` — target-side links only |
| `part_of` | no | `true` makes the target a constituent part of this root |
| `settles` | no | owning links only (`part_of` / `on_delete: "cascade"`): `false` keeps the ownership but exempts the target from its owner's terminal-state lock — notes, comments, messages ([state-machines.md](state-machines.md#a-terminal-state-settles-the-record)) |

The most common mistake is `via`: a `customer` with many `task` rows declares
`via: "customer_id"` because `customer_id` lives on `task`. Declare (and
refine) the target before the link that points at it.

**Anchor every child.** A table with an FK column declares the upward
belongs-to link, and the parent declares the downward one with the delete
policy:

```json
{
  "name": "task",
  "links": {
    "customer": { "target": "customer", "cardinality": "one", "via": "customer_id", "fk_on": "source" }
  }
}
```

```json
{
  "name": "customer",
  "links": {
    "tasks": { "target": "task", "cardinality": "many", "via": "customer_id", "on_delete": "cascade" },
    "orders": { "target": "order", "cardinality": "many", "via": "customer_id", "on_delete": "restrict" }
  }
}
```

Validation per link: the `target` is registered (`Link "tasks": target
ObjectType "task" is not registered`), the `via` column exists on the FK side
(`Link "tasks": via column "customer_id" not found on target table "task"`),
and a source-side link carries no `on_delete` (`on_delete_shape_invalid`).

## `on_delete`

Data semantics, not handler semantics: declared once on the target-side link,
honoured by **every** delete path — the generated `<type>.delete`, a handler
`delete` step, and `objects.delete`.

| Policy | On parent delete | Use for |
|---|---|---|
| `restrict` (default) | refuse with `INVALID_STATE` naming the link and child count | history you must not lose (orders under a customer) |
| `cascade` | delete children first, depth-first, same transaction; nested cascades chain | parts meaningless without the parent (lines under an order) |
| `set_null` | null the `via` column on every child | children that outlive the parent (tasks of a removed assignee) |

`set_null` needs a nullable `via` column (refine the target property
`nullable: true` first). A source-side belongs-to link (`fk_on: "source"`) is
navigational only: it must omit `on_delete`; put the policy on the reverse link.

A link to a **platform** ObjectType (`org_member`, `_audit_event`,
`signature_request`, …) may navigate but may not carry `on_delete` or
`part_of` (`platform_type_verbs_refused`) — both would let your org delete or
write platform rows. The default `restrict` still refuses your delete while
platform rows reference the row.

## Rollups

`rollups` on any type: result name → `{ link, agg, column?, label?, type? }`
over one of its to-many links (the FK on the child). `agg` is `count` ·
`sum` · `avg` · `min` · `max` · `exists` (a boolean: at least one linked row
the reader may read); `exists` takes no `column`, `count` needs none, and the
rest need a numeric child `column` (mapped or derived).

```json
{ "name": "client", "links": { "analyses": { "target": "analysis", "cardinality": "many", "via": "client_id" } },
  "rollups": { "analysis_count": { "link": "analyses", "agg": "count" }, "has_analyses": { "link": "analyses", "agg": "exists" } } }
```

Each read computes it over the child rows the reader may read (row filters
and walls apply), so two readers can see different figures; it is withheld,
like a masked column, when the reader may not `aggregate` the child or read
its FK or the summed column. It rides on `get`, `list`, search hits, `navigate`, `--expand` and a
handler ref to a row a step read (`$claim.open_lines`; withheld: `FORBIDDEN`) — counting only the rows the CALLER reads, so a guard over every row (deposit paid, overdue, instalments) is a `read_aggregate`, which counts them all; lists and views filter (`exists` takes `true` /
`false`), sort and preset on it; an action's `showWhen` (and `display.actions`) names it like a property, `inputShowWhen` as `$subject.<rollup>` (withheld: the action is offered, the input asked). A to-one link (use a
[lookup](object-property-types.md#lookup-properties)), an unknown link,
`exists` with a `column` or a non-numeric column is `rollup_invalid`.

**Rollup or derived?** A filter on a rollup works on every read of a block
(search, totals, group and lane counts, metric, chart, grouped table). As a
column it is refused elsewhere — an aggregate's measure or group-by, a search
match, a handler `where` / `order_by`, a row filter — with
`rollup_not_compiled`, naming it and the fix (aggregate the link's rows), at
declare and at read; a table `totals` refuses it (`invalid_table_totals`).
When the figure must total or chart,
declare the same sum as a derived property (a correlated subquery,
[object-property-types.md](object-property-types.md#derived-properties)); it
sums every child row, so a row-scoped reader is not shown it
([access](access-policies.md#derived-properties-under-access)).

## Compound objects

A **compound** is a root whose identity includes its parts — an invoice and its
lines. `part_of` is distinct from `on_delete`: `cascade` answers "what happens
when the parent is deleted", `part_of` answers "is this child part of the
parent's identity".

```json
{
  "name": "invoice",
  "links": {
    "lines": { "target": "invoice_line", "cardinality": "many", "via": "invoice_id", "on_delete": "cascade", "part_of": true }
  },
  "rollups": {
    "total": { "link": "lines", "agg": "sum", "column": "amount" },
    "line_count": { "link": "lines", "agg": "count" }
  },
  "parts_frozen_when": { "column": "status", "in": ["sent", "paid"] }
}
```

### `part_of` rules (checked at `catalog.apply` and at registration)

| Rule | Violated by | Code |
|---|---|---|
| target-side FK only | `part_of` on an `fk_on: "source"` link | `part_of_shape_invalid` |
| no orphaning | `part_of` with `on_delete: "set_null"` (pair it with `cascade` or `restrict`) | `part_of_shape_invalid` |
| NOT NULL FK | the child's `via` column is nullable | `part_of_shape_invalid` |
| target exists | unknown target | `part_of_unknown_target` |
| forest | a child with two `part_of` parents, or any cycle (self-loop included) | `part_of_forest_violated` |
| not a platform type | target is a platform ObjectType | `platform_type_verbs_refused` |

Parts of parts are fine (a three-level tree).

### `parts_frozen_when`

`{ column, in: [...] }` on the root (`column` a mapped root property, `in`
non-empty). While the root's value is in the set, any insert/update/delete of a
part is refused with `CONFLICT` / `details.reason: "FROZEN"`. Creating a root
that is born frozen still succeeds. A non-compound type, an unmapped column or
an empty set → `parts_frozen_invalid`.

### What a compound changes

Reading: `canon objects get invoice inv_… --expand` (REST
`GET /v1/objects/invoice/<id>?expand=1`) returns the root with its parts
nested. The generated verbs change too (`<root>.create` takes nested parts,
`<root>.replace` appears, part writes go through the root's version anchor and
freeze gate) — see [tables-and-import.md](tables-and-import.md).
</content>
