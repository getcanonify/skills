---
name: canonify-objects-actions
description: The reference for Canonify's catalog mechanics — tables and auto-CRUD, bulk import, ObjectTypes (properties, mapped/derived, enums, display, visibility, temporal), links and compound objects, ActionTypes (input, governance, idempotency, presentation), the handler DSL, state machines, schedules and reactions, and the built-in e-signature and outbox actions.
version: 2.0.0
---

# Canonify — Objects & Actions

The one reference for what you declare in a catalog. Read the base `canonify`
skill first for the authoring loop, auth, the envelope and errors
(`canon skill --name canonify instructions`). Each fact below has one home;
the resources are terse references — open the one you need.

## The mental model

- A **table** (`schema.propose_migration`) gets a default ObjectType and the
  generated CRUD actions at once. Load rows in bulk with `objects.import`.
- An **ObjectType** (`catalog.refine_object_type`) is the typed shape over a
  table: `mapped` properties (real columns) and `derived` ones (read-only SQL
  expressions), ten property types, enum `values` with colours and labels,
  `display` (titles, forms, lists), `hidden` columns and `confidential` types,
  and `temporal` effective dating.
- A **link** is a typed edge with a `cardinality` (`one`/`many`), a `via` FK
  column and, on the parent side, an `on_delete` policy honoured by every delete
  path. `part_of` links make compound objects.
- An **ActionType** (`actions.declare`) is a governed write: an `input`
  descriptor, `governance` (who, approval), `idempotency`, an optional
  `transition` marker, `presentation`, and a **handler** — typed steps
  (`read`, `read_optional`, `read_many`, `read_aggregate`, `insert`, `update`,
  `delete`, `assert`, `transition`, `call`, `for_each`) glued by refs
  (`$input.<field>`, `$<step>.<field>`, `$now`, `$principal.id`).
- A **state machine** (`state_machines.register`) owns an enum property: only
  actions carrying its transition marker move it, with a guard, an automatic
  state write and hooks.
- Everything is isomorphic: one declaration is the same action on REST, CLI and
  MCP, returning the same envelope.

## Resources

| Page | The reference for |
|---|---|
| [tables-and-import.md](resources/tables-and-import.md) | `schema.propose_migration`, column conventions, what auto-CRUD generates (incl. compound roots), generated-update semantics, retyping columns, `objects.import` |
| [object-property-types.md](resources/object-property-types.md) | `catalog.refine_object_type`: property kinds, the ten types, the JSON/boolean wire contract, enum narrowing, the `valueColors` tokens, `valueLabels`, `currency`, labels, `display`, `hidden`/`confidential`, `temporal`, re-refine hashes |
| [object-links-and-cardinality.md](resources/object-links-and-cardinality.md) | links, `on_delete`, `part_of`, `rollups`, `parts_frozen_when` |
| [actions-input-governance.md](resources/actions-input-governance.md) | `actions.declare`: every field, the input-type table, `subject`, governance and `operation`, unattended authority, `target`, idempotency, `presentation`, `actions.update` |
| [actions-handler-patterns.md](resources/actions-handler-patterns.md) | the handler DSL: refs, steps, `lead.convert`, predicates, `where`, nullable bindings, expressions, aggregates, sets, fragments, `on_fail`, type-check errors |
| [state-machines.md](resources/state-machines.md) | `state_machines.register`, what a transition marker buys, set transitions, the bound property |
| [schedules-and-org-admin.md](resources/schedules-and-org-admin.md) | schedules and reactions, `org.*` lifecycle actions |
| [esign-actions.md](resources/esign-actions.md) | the built-in e-signature actions |
| [outbox-actions.md](resources/outbox-actions.md) | `outbox.inspect` / `outbox.requeue` |

Patterns that combine these (a sequential human code, create-then-transition)
live in `canonify-app-authoring`'s
[patterns.md](../canonify-app-authoring/resources/patterns.md); modelling
obligations in [`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md).

## Read this before your first declaration

- Declare in order: tables → ObjectType refinements (enums before machines,
  link targets before links) → actions → state machines → views.
- Narrow every state column to an `enum` before binding a machine to it; the
  narrowing is checked against the rows already stored.
- Money is `type: money` — integer minor units with a `currency`.
- `via` names the FK column on the **target** table unless `fk_on: "source"`.
- A handler never writes a machine-bound property; the marker does
  (`direct_state_write`).
- Declarations are idempotent per key: re-sending identical JSON is a no-op,
  changing one needs `actions.update` / a re-refine with `expectedHash`.
- `canon actions describe <name> --json` and `canon schema describe --object
  <type>` show what is live; when this skill and the live schema disagree, the
  live schema wins.
</content>
