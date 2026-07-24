# Master-detail recipe

A list that drills into one record's full detail. This is the most common
two-block layout: a `table` master on top, a `detail` below it driven by the
render context.

---

## The pattern

A `composite` holds two children:

1. A `table` — the master list. Selecting a row puts its id into the render
   context.
2. A `detail` — reads that record id via `$context` and shows the full
   property grid.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "table",
      "source": { "objectType": "customer", "sort": "name" },
      "columns": [
        "name",
        { "ref": "email", "label": "Email", "sortable": true },
        "tier",
        "appointments.count"
      ]
    },
    {
      "type": "detail",
      "source": { "objectType": "customer" },
      "properties": [
        "name",
        "email",
        { "ref": "tier", "label": "Loyalty tier" },
        "phone",
        "notes"
      ]
    }
  ]
}
```

---

## Field-by-field detail mapping

The `detail` block's `properties` array is an explicit, ordered map of which
properties appear and how they're labelled. Each entry is either a bare
property name (uses the ObjectType's default label) or a `{ ref, label }`
object that overrides the label. Omit `properties` entirely to fall back to
the ObjectType's default grid. A `detail` also accepts an optional `header`
band (title / subtitle / status chip / avatar) — see
[viewspec-block-types §2](viewspec-block-types.md).

```
properties: [ "name", "email", {ref:"tier", label:"Loyalty tier"}, "phone", "notes" ]
                |        |             |                              |        |
                v        v             v                              v        v
            +-----------------------------------------------------------------+
            |  Name          Acme Corp                                        |
            |  Email         ops@acme.test                                    |
            |  Loyalty tier  Gold                                            |
            |  Phone         +45 12 34 56 78                                  |
            |  Notes         Renewal due Q3                                   |
            +-----------------------------------------------------------------+
```

---

## ASCII layout

```
+----------------------------------------------------------+
|  Customers                                  [ filter ▾ ]  |   <- table (master)
+-------------+---------------------+-------+--------------+ |
|  Name       |  Email              | Tier  | Appts (#)    | |
+-------------+---------------------+-------+--------------+ |
| ▶ Acme Corp |  ops@acme.test      | Gold  |    12        | |   selected row
|   Beta Ltd  |  hi@beta.test       | Std   |     7        | |   -> $context.id
+-------------+---------------------+-------+--------------+ |
                                                            |
+----------------------------------------------------------+
|  Customer                                                 |   <- detail
+----------------------------------------------------------+   ($context.customer.id)
|  Name          Acme Corp                                  |
|  Email         ops@acme.test                              |
|  Loyalty tier  Gold                                       |
|  Phone         +45 12 34 56 78                            |
|  Notes         Renewal due Q3                             |
+----------------------------------------------------------+
```

When no record id is in context, the detail renders a clear empty state
("needs a record id") rather than a broken page.

---

## Variant: detail with an embedded child table

A detail page often wants a related list scoped to the record — e.g. a
customer's appointments. Nest a scoped `table` after the detail. The child
table filters on `$context.id`, so it only shows rows for the record in
scope.

```json
{
  "type": "composite",
  "children": [
    {
      "type": "detail",
      "source": { "objectType": "customer" },
      "properties": ["name", "email", "tier"]
    },
    {
      "type": "table",
      "source": {
        "objectType": "appointment",
        "filter": { "customer_id": "$context.id" },
        "sort": "scheduled_at"
      },
      "columns": ["scheduled_at", "status", "notes"]
    }
  ]
}
```

```
+--------------------------------------+
|  Customer                            |
|  Name   Acme Corp                    |
|  Email  ops@acme.test                |
|  Tier   Gold                         |
+--------------------------------------+
|  Appointments (this customer)        |   <- scoped child table
|  When         | Status   | Notes     |
|  2026-06-10   | booked   | annual    |
|  2026-07-02   | tentative| follow-up |
+--------------------------------------+
```

Note the child table above binds a **different** ObjectType (`appointment`)
than the composite's own (`customer`). That is allowed and intentional — see
below.

---

## Compound views — a child on a different ObjectType

A composite child may declare its own `source.objectType` that **differs** from
the view's top-level `object_type`. This is how you build a *compound* view: one
parent record plus a child table of related rows on a second ObjectType — e.g.
an **invoice** detail above an **invoice_line** table.

```json
{
  "name": "Invoice.detail",
  "object_type": "invoice",
  "spec": {
    "type": "composite",
    "children": [
      {
        "type": "detail",
        "source": { "objectType": "invoice" },
        "properties": ["number", "issued_at", "total"]
      },
      {
        "type": "table",
        "source": {
          "objectType": "invoice_line",
          "filter": { "invoice_id": "$context.id" }
        },
        "columns": ["description", "qty", "unit_price", "line_total"]
      }
    ]
  }
}
```

The rule: for a **top-level** `table` / `detail` / `board` block, the
declaration's `object_type` must equal the block's `source.objectType` (a
mismatch fails with `source_object_type_mismatch`). But for a **composite
child**, declare resolves the child's `source.objectType` independently:

- If the child's `source.objectType` equals the top-level `object_type`, it
  validates against that — the ordinary case.
- If it *differs*, declare looks the child's ObjectType up in the live registry
  and validates the child's columns / filters against **that** ObjectType
  instead. The child binds its own type cleanly.
- `source_object_type_mismatch` still fires here only when the child names an
  ObjectType that **does not exist** in the registry — the child claims a type
  that isn't real.

So a child on a real, different ObjectType is fine; a child on a *non-existent*
ObjectType is the mismatch. Scope the child to the parent row the usual way —
a `$context.id` filter on the child's foreign-key property (`invoice_id` above).

**A bare child table auto-scopes to the record.** You don't strictly *need* the
explicit `filter` above: inside a **record-scoped composite** (a `composite`
whose view carries its own `object_type`), a child `table` with **no declared
filter** is auto-scoped to the current record — the renderer finds the **one**
`cardinality: 'many'` link from the parent type to the child type and filters on
that link's `via` FK column. Two guard rails keep it safe:

- If **more than one** many-link connects the two types, the scope is
  **ambiguous** — the renderer scopes nothing and warns; add an explicit
  `source.filter` to disambiguate.
- If you already key the child's `filter` on one of those FK columns, **you own
  the scoping** — the renderer doesn't AND in a second inferred key.

A child type the parent has **no** many-link to is left **org-wide** inside the
record page — which looks like a scoping bug. That gap is caught at author time
by the `elegance_unlinked_child_table` advisory (see the `canonify` skill's
error-codes): declare the missing many-link, or add an explicit filter.

---

## Where a row click lands — declared detail views

By default, clicking a row navigates to the **auto record route**
(`/app/$slug/record/$table/$id`) — the generated `AutoDetail` page. When you
**declare a curated record view** for that row's ObjectType — a `detail`, or a
`composite` containing a `detail` child — a row click instead resolves to that
view over the **record-scoped view route** (`/app/$slug/view/$viewId/$id`). So
declaring `Customer.account` (a composite over `customer`) is what makes clicking
a customer row open your curated account page instead of the inferred one. No new
vocabulary — declaring the view *is* the opt-in.

Resolution rule (a deterministic platform default):

1. **App scope (type-level gate).** With an app active, a row click only reroutes
   for object-types the app's **nav** actually surfaces (the same scope that
   gates Browse-data). A type the app doesn't surface stays on the auto route.
   (This is a client-side routing convenience, not a data boundary — the resolved
   view still fetches through server-side policy.)
2. **Candidates** = every declared view whose `object_type` matches the row's
   type **and** which can render a single record (a `detail`, or a `composite`
   with a `detail` child — a `table`/`board`/`queue`/`metric` renders a list, so
   it is never a candidate). No candidate ⇒ fall back to the auto route.
3. **Deterministic tiebreak** among candidates: a view **in the current app's
   nav** wins over one that isn't; final tiebreak is **ascending view name**
   (`localeCompare`) — a total, stable order, so the choice never flickers.

An ObjectType rich enough to deserve a curated page (three or more many-links)
but with **no** declared detail/composite anywhere draws the
`elegance_no_detail_view` advisory — its rows land on the inferred page instead
of an authored one.

**On the record-scoped view route**, a `composite` behaves as a single record
page rather than a stack of independent views:

- **One toolbar per composite.** Non-owner `detail` children suppress their own
  action toolbar, so the record page shows a single action toolbar (the owner's),
  not one per child.
- **Child tables auto-scope** to the record via the many-link FK, as above.
- **Header refs traverse links.** A declared `header` whose `title` / `subtitle` /
  `status` / `avatar` follows a link (`organization.name`, or a `{organization.cvr}`
  subtitle template) resolves against the **linked** record — the header reads a
  traversal exactly the way the grid columns do, instead of rendering an em-dash.
