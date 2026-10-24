---
name: canonify-app-authoring
description: A top-to-bottom tutorial for building a Canonify app declaratively — the model-b authoring loop end to end, five chapters, one declaration verb each. Read this after the base `canonify` skill.
version: 2.0.0
---

# Canonify App Authoring — build an app from nothing

**Start here for screens:** the [ViewSpec cheatsheet](../canonify-viewspecs/resources/viewspec-cheatsheet.md)
— every block, field and common declare error on one page.

This is a **tutorial**: it takes an empty org to a working app, one declaration
verb per chapter, and links to the one reference page for each verb instead of
restating it. The example is a small CRM (`customer`, `contact`, `deal`); the
nouns are illustrative. Invocation (CLI / REST / MCP, the envelope, `--org`) is
in the base [`canonify`](../canonify/SKILL.md) skill — this page shows the CLI
form only.

**Want the whole thing at once?** [resources/worked-example.md](resources/worked-example.md)
is a complete small app as one bundle (tables, ObjectTypes, a state machine,
five actions, seven views, an app, a scenario) that passes `canon catalog test`.

## The authoring loop

| Chapter | Verb | You declare | Reference |
|---|---|---|---|
| 1 | `schema.propose_migration` | Tables; auto-CRUD adds a default ObjectType and the CRUD actions | [tables-and-import.md](../canonify-objects-actions/resources/tables-and-import.md) |
| 2 | `catalog.refine_object_type` | Enums, money, links, titles, labels | [object-property-types.md](../canonify-objects-actions/resources/object-property-types.md), [object-links-and-cardinality.md](../canonify-objects-actions/resources/object-links-and-cardinality.md) |
| 3 | `state_machines.register` | A lifecycle bound to an enum property | [state-machines.md](../canonify-objects-actions/resources/state-machines.md) |
| 4 | `actions.declare` | Custom ActionTypes (the handler DSL) | [actions-input-governance.md](../canonify-objects-actions/resources/actions-input-governance.md), [actions-handler-patterns.md](../canonify-objects-actions/resources/actions-handler-patterns.md) |
| 5 | `view_specs.declare` | How the app renders | [canonify-viewspecs](../canonify-viewspecs/SKILL.md) |

A read-only dashboard may need only Chapters 1 and 5. **Invoke in this order:
tables → refinements → actions → state machines → views.** The chapters teach
the machine (3) before the actions (4), but `state_machines.register` refuses a
transition `via` or hook action that is not declared yet (`unknown_action`,
`unknown_hook_action`), while an action's `transition` marker may name a machine
registered later. `canon catalog apply` orders a bundle for you. The five
chapters, in invocation order, are one runnable scenario:
`canon catalog test resources/tutorial-replay`
([session.yaml](resources/tutorial-replay/session.yaml)).

**Agent-authored declarations park for approval.** An agent's `actions.declare`,
`catalog.refine_object_type` or `view_specs.declare` returns `approval_status:
'pending'` and a `request_id`; nothing exists until a `manage` holder (an owner)
runs `actions.approve { request_id }` (or `actions.reject`), which re-dispatches
it. `canon catalog apply` does not park: it is one `manage` action, so an agent
key with `manage` applies a bundle directly.

## Before you start: discover

```sh
canon actions list --json     # every action — platform + yours
canon objects list-types      # every ObjectType
canon schema describe         # the data-model overview
```

Run them once **in a fresh org** too: what comes back there is the platform
surface (e-signature, `file` storage, audit, outbox, approvals, idempotency,
state machines) — never build a second copy. Then read
[`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md) before your
first migration: its four obligations are nearly impossible to retrofit.

When something fails: `canon audit <audit_event_id>` shows the full envelope;
file a platform bug with `canon report "<command>" "<error>"`.

## Chapter 1 — Tables

```sh
canon actions invoke schema.propose_migration --input - <<'JSON'
{ "description": "customer table — the CRM root entity",
  "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'prospect', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)" }
JSON
canon actions list --json | jq -r '.[].name' | grep '^customer\.'   # create read update delete list
```

Add `contact (…, customer_id TEXT NOT NULL, …)` and `deal (…, customer_id TEXT
NOT NULL, amount INTEGER NOT NULL, stage TEXT NOT NULL DEFAULT 'open', won_at
TEXT, …)` the same way. Money is an `INTEGER` of minor units (`125000` =
1,250.00); timestamps are ISO 8601 `TEXT` (no zone = the org's clock). Column
conventions and what auto-CRUD generates: [tables-and-import.md](../canonify-objects-actions/resources/tables-and-import.md).

## Chapter 2 — Refine the ObjectType

The default ObjectType is permissive (every column a `mapped` property typed from
SQL, no enums, links or money). Narrow the state column to an `enum` — a state
machine can only bind to one — declare money with its currency, and anchor the
child:

```yaml
action: catalog.refine_object_type
input:
  name: deal
  description: A sales opportunity against a customer — Deal.stage FSM-governed.
  idPrefix: dl
  properties:
    stage:  { kind: mapped, column: stage, type: enum, values: [open, proposal_sent, won, lost] }
    amount: { kind: mapped, column: amount, type: money, currency: DKK }
    won_at: { kind: mapped, column: won_at, type: timestamp, nullable: true }
  links:
    customer: { target: customer, cardinality: one, via: customer_id, fk_on: source }
```

On `customer`, declare the reverse links with their delete policy:
`contacts: { target: contact, cardinality: many, via: customer_id, on_delete: cascade }`
and `deals: { …, on_delete: restrict }`. Then `canon objects navigate customer
cus_… deals` works. Every refinement field (titles, labels, `valueColors`,
`display`, derived and lookup properties, `temporal`):
[object-property-types.md](../canonify-objects-actions/resources/object-property-types.md);
links and `on_delete`: [object-links-and-cardinality.md](../canonify-objects-actions/resources/object-links-and-cardinality.md).

## Chapter 3 — Bind a state machine

```yaml
action: state_machines.register
input:
  kind: state_machine
  track: customer
  name: Deal.stage
  objectType: deal
  property: stage
  initial: open
  states:
    open:          { label: Open }
    proposal_sent: { label: Proposal sent }
    won:           { label: Won, terminal: true }
    lost:          { label: Lost, terminal: true }
  transitions:
    - { via: deal.send_proposal, from: [open],                to: proposal_sent }
    - { via: deal.win,           from: [proposal_sent],       to: won }
    - { via: deal.lose,          from: [open, proposal_sent], to: lost }
  on_enter:
    won:
      - action: deal.activate_customer
        inputs: { deal_id: $row.id }
```

An action whose `transition` marker matches gets a `from` guard
(`INVALID_STATE`), the state write after its handler (a handler that writes
`stage` itself is refused with `direct_state_write`), and the hooks — for free.
`canon state-machines render Deal.stage` draws it. Reference:
[state-machines.md](../canonify-objects-actions/resources/state-machines.md).

## Chapter 4 — Declare actions

```json
{
  "name": "deal.win",
  "description": "Mark a deal as won.",
  "object_type": "deal",
  "input": { "deal_id": { "type": "string" } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent", "service_account"] },
  "idempotency": "none",
  "transition": { "state_machine": "Deal.stage", "from": ["proposal_sent"], "to": "won" },
  "handler": {
    "steps": [
      { "op": "update", "object": "deal", "where": { "id": "$input.deal_id" }, "set": { "won_at": "$now" } }
    ],
    "returns": { "deal_id": "$input.deal_id" }
  }
}
```

```sh
canon actions invoke actions.declare --input @deal-win.json
canon deal.win --deal_id dl_…            # per-action sugar, live immediately
```

`deal.send_proposal` and `deal.lose` are the same shape; a transition with no
other effect is one `read` of the row (a handler needs at least one step, and the
read turns a bad id into `NOT_FOUND`). `deal.activate_customer` — the hook above
— reads the deal, `assert`s it is `won`, and updates its customer to `active`,
all in one transaction (full text in the tutorial replay). Re-declaring
identical JSON is a no-op; changed JSON needs `actions.update`. Its
`presentation` builds the form, which reads the record it opens from:
`inputDefaults: { "owner_id": "$subject.owner_id" }` on a `deal.reassign`
opens on the deal's current owner, and an `inputFilters` scope may read
`$subject.<column>` too. Every field:
[actions-input-governance.md](../canonify-objects-actions/resources/actions-input-governance.md);
every step and recipe: [actions-handler-patterns.md](../canonify-objects-actions/resources/actions-handler-patterns.md);
codes, create-then-transition, bookings and weekly series: [resources/patterns.md](resources/patterns.md).

## Chapter 5 — Compose views

```json
{
  "name": "Customer.list",
  "object_type": "customer",
  "spec": { "type": "table", "source": { "objectType": "customer" }, "columns": ["name", "email", "status", "deals.count"] }
}
```

```sh
canon actions invoke view_specs.declare --input @customer-list.json
```

`name` is dot-separated; `spec.source.objectType` must equal `object_type`;
every column must resolve (declare-time checked). `deals.count` is a link
aggregate (`deals.sum:amount` for a sum). Don't stop at a bare table: a
`detail` header, `valueColors`, filter `presets` and curated columns are
what make a screen read well — the recipes in
[`canonify-viewspecs`](../canonify-viewspecs/SKILL.md) start from those.

## Fill it with data

Load realistic data before you judge a screen, with `objects.import` (up to 500
rows per atomic call), not a loop of creates. **Keep a seed's dates and pin the
org clock to the seed's today** (`--demo_now`, owner only): `$now`, `:today`,
derived `:now` / `:today`, Today lanes, ages and queue buckets then
read that day. Set `demo_now` to `null` at go-live (real time again).

```sh
canon org update --timezone Europe/Copenhagen --demo_now 2026-10-23T09:30
canon objects import customer --file customers.jsonl
canon objects import deal     --file deals.jsonl  # parents first
```

Import is the one path for history: a row may carry its own `id`,
`created_at` and any declared state, attachments their uploader and time, and
no hook runs. Details:
[tables-and-import.md](../canonify-objects-actions/resources/tables-and-import.md).

## Group it, crystallize it

Put the views in an App (`apps.declare`: nav, `defaultView`, Browse data, count
badges), then `canon catalog export` the canon to a bundle, commit it, and from
then on `validate` → `apply` it. Both are in
[resources/catalog-bundle.md](resources/catalog-bundle.md). For two languages:
[resources/bilingual-apps.md](resources/bilingual-apps.md).

## See it in the app

The CLI proves the data, not the screen. After any view, nav or App change, open
the app and look: the columns, the nav, the "Browse data" sidebar (scoping and
`part_of` nesting). An agent trades its API key for a short-lived web session
(the key never enters the browser) and drives the app with `agent-browser`:

```sh
SESSION=$(curl -s -X POST https://api.canonify.app/api/auth/agent/exchange \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Origin: https://app.canonify.app" | jq -r .token)
agent-browser open https://app.canonify.app
printf "localStorage.setItem('canonify_token', %s); 'ok'" "\"$SESSION\"" | agent-browser eval --stdin
agent-browser open https://app.canonify.app/app/<your-app-slug>
agent-browser screenshot /tmp/my-app.png
```

You appear with an **"Agent" badge**, scoped to the key's one org (no org
switcher or account settings). Re-exchange when the session expires; revoke the
key with `canon key revoke <id>`. To see what a member sees, exchange a key
minted `--role member --groups <g>` instead
([access policies](../canonify-objects-actions/resources/access-policies.md#see-it-as-a-member));
a member runs every custom action you did not
[gate to a group](../canonify-objects-actions/resources/access-policies.md#gate-an-action-to-a-group).
Start with `agent-browser skills get core --full`.
Charts and metrics load after the page: wait a few seconds and take viewport
screenshots — a full-page capture (`screenshot --full`) can show them empty.

## You shipped a slice, not an app

One domain, one nav is a **slice**. The five verbs are cheap to repeat, and
repeating them across every domain of a big brief yields many prototypes and no
product. Before the next domain, read
[`canonify-app-lifecycle`](../canonify-app-lifecycle/SKILL.md); prove the slice
with a scenario ([`canonify-scenarios`](../canonify-scenarios/SKILL.md)).

## Resources

- [resources/worked-example.md](resources/worked-example.md) — a complete app bundle and its scenario
- [resources/patterns.md](resources/patterns.md) — a human code (`J-1042`); create-then-transition; bookings (free slots, a weekly series, Today, a waitlist, claims) with `booking-example/`
- [resources/catalog-bundle.md](resources/catalog-bundle.md) — the App record and the bundle lifecycle (export, validate, apply, scratch)
- [resources/bilingual-apps.md](resources/bilingual-apps.md) — `supported_locales`, `@t:` keys, `i18n/<locale>.json`
- [resources/tutorial-replay/session.yaml](resources/tutorial-replay/session.yaml) — this tutorial as one runnable scenario
