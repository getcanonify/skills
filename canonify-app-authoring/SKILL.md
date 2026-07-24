---
name: canonify-app-authoring
description: A top-to-bottom tutorial for building a Canonify app declaratively — the model-b authoring loop end to end, five chapters, one declaration verb each. Read this after the base `canonify` skill.
version: 1.0.0
---

# Canonify App Authoring — build an app from nothing

This is a **tutorial**, not a reference. It walks you — an AI agent — from
an empty Canon org to a working application, one declaration verb per
chapter, using a single worked example you can adapt. When you want the
full contract for any one verb (every field, every failure code), jump to
the base [`canonify`](../canonify/SKILL.md) skill or the per-chapter
resource file linked at the end of each section.

Everything here is **declarative**. You never write code. You build the
app by invoking platform Actions through one of three isomorphic surfaces
— REST, CLI, or MCP — and the runtime parses your declarations, runs them
inside a transactional frame (audit + outbox + idempotency + governance +
state-machine guards all baked in), and surfaces a uniform envelope. The
same call produces a byte-identical envelope on all three surfaces; that
is the conformance gate, and it means **you never have to think about
which surface you are on**.

The example domain in this tutorial is a small CRM — `customer`,
`contact`, `deal`. These nouns are **illustrative only**; they are not
product commitments. Swap them for whatever your app models (`invoice`,
`ticket`, `shipment`, `patient`) and the shape of every step stays the
same.

## The authoring loop

Building any Canonify app is the same five moves, in order:

| Chapter | Verb (Action) | You declare | Resource |
|---|---|---|---|
| 1 | `schema.propose_migration` | Tables. Auto-CRUD gives you a default ObjectType + five CRUD actions for free. | [schema-bootstrap.md](resources/schema-bootstrap.md) |
| 2 | `catalog.refine_object_type` | The rich domain shape — enums, links, descriptions, hidden columns. | [objecttype-refinement.md](resources/objecttype-refinement.md) |
| 3 | `state_machines.register` | A finite-state machine bound to an enum property. | [state-machines-binding.md](resources/state-machines-binding.md) |
| 4 | `actions.declare` | Custom multi-step ActionTypes (the handler DSL). | [actions-multi-step.md](resources/actions-multi-step.md) |
| 5 | `view_specs.declare` | How the app renders — tables, details, panels, metrics. | [viewspec-quick-start.md](resources/viewspec-quick-start.md) |

You do not have to do all five for a trivial app — a read-only dashboard
might be Chapters 1 + 5 only. But the order matters: each verb references
artefacts the earlier verbs created. A StateMachine needs an enum
property to bind to; a transition action needs its StateMachine; a
ViewSpec column ref needs its ObjectType property to exist.

## Before you start: look at what's already there

Your **first command in any session** should be discovery. Never declare
blind — the org may already have tables, ObjectTypes, and actions.

```sh
canon actions list --json     # every action — platform + custom
canon objects list-types      # every ObjectType
canon schema describe         # the data-model overview
```

Over REST that is `GET /v1/actions`, `GET /v1/objects/_types`,
`GET /v1/schema_describe`. Over MCP it is the `actions.list` tool. All
three return the same catalog.

If anything in this tutorial fails in a way you can't explain, capture it
with `canon audit <audit_event_id>` (every call — success, denial, or
error — writes an audit row) and, if it looks like a platform bug, file
it with `canon report "<command>" "<error>"`.

---

## Chapter 1 — Bootstrap the schema (`schema.propose_migration`)

A table is one Action call. Auto-CRUD generates the default ObjectType and
the five CRUD ActionTypes (`<table>.create`, `.read`, `.update`,
`.delete`, `.list`) from the table shape — so a single migration already
gives you a working, queryable data model on every surface.

**Action:** `schema.propose_migration`
**Input shape:** `{ description: string, sql: string }`
**Envelope on success:** `{ data: { migration_id, ... }, audit_event_id, approval_status: 'not_required' }`

Start with the parent table for your example CRM:

```sql
CREATE TABLE customer (
  id          TEXT PRIMARY KEY,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  status      TEXT NOT NULL DEFAULT 'prospect',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
)
```

Conventions (from `AGENTS.md`): `id TEXT PRIMARY KEY` holds a prefixed
ULID (`cust_…`); `created_at` / `updated_at` are `TEXT NOT NULL` ISO 8601
UTC, never integer epochs; tables prefixed with `_` are platform-owned —
do not declare them.

### Invoke it — three surfaces, one call

**CLI** (meta-tool form; `--input -` reads JSON from stdin, `@file.json`
reads a file):

```sh
canon actions invoke schema.propose_migration --input - <<'JSON'
{
  "description": "customer table — the CRM root entity",
  "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'prospect', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
}
JSON
```

**REST** (`POST /v1/actions/<name>` with a `{ input, idempotency_key? }`
body; authenticate with `Authorization: Bearer <api_key>`):

```sh
curl -X POST https://api.canonify.app/v1/actions/schema.propose_migration \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
      "description": "customer table — the CRM root entity",
      "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT '\''prospect'\'', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
    }
  }'
```

**MCP** (the `actions.invoke` tool — args are `{ name, args }`):

```json
{
  "name": "actions.invoke",
  "arguments": {
    "name": "schema.propose_migration",
    "args": {
      "description": "customer table — the CRM root entity",
      "sql": "CREATE TABLE customer (id TEXT PRIMARY KEY, name TEXT NOT NULL, email TEXT NOT NULL, status TEXT NOT NULL DEFAULT 'prospect', created_at TEXT NOT NULL, updated_at TEXT NOT NULL)"
    }
  }
}
```

After it commits, the new CRUD actions are live immediately — no restart:

```sh
canon actions list --json | jq '.[].name' | grep '^customer\.'
# customer.create
# customer.read
# customer.update
# customer.delete
# customer.list
```

Repeat for `contact` and `deal` so later chapters have something to link
and transition:

```sql
CREATE TABLE contact (
  id          TEXT PRIMARY KEY,
  customer_id TEXT NOT NULL,
  name        TEXT NOT NULL,
  email       TEXT NOT NULL,
  role        TEXT,
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);

CREATE TABLE deal (
  id          TEXT PRIMARY KEY,
  customer_id TEXT NOT NULL,
  title       TEXT NOT NULL,
  amount      TEXT NOT NULL,
  stage       TEXT NOT NULL DEFAULT 'open',
  created_at  TEXT NOT NULL,
  updated_at  TEXT NOT NULL
);
```

Full contract — what auto-CRUD generates, how to override defaults,
failure modes — in [schema-bootstrap.md](resources/schema-bootstrap.md).

---

## Chapter 2 — Refine the ObjectType (`catalog.refine_object_type`)

Auto-CRUD's default ObjectType is deliberately permissive: every column is
a `mapped` `string`, no links, no description. Refine it into the **rich**
domain shape. This is the chapter that turns "a table you can CRUD" into
"a typed object the platform understands".

**Action:** `catalog.refine_object_type`
**Input shape:** `{ name: string, description?: string, properties?: {...}, links?: {...} }`
**Envelope on success:** `{ data: { name, refined: true }, audit_event_id, approval_status: 'not_required' }`

The load-bearing pattern is **enum narrowing** — a state column must be a
typed `enum` before a StateMachine can bind to it in Chapter 3. Refine
`deal.stage` and add the typed links between your three ObjectTypes:

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
  links:
    customer: { target: customer, cardinality: one, via: customer_id }
```

And on the parent, declare the reverse links with `on_delete` policies:

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

`on_delete` policies are honoured by **every** delete path, not just
declared handlers: `restrict` (default — `INVALID_STATE` if children
exist), `cascade` (delete children first), `set_null` (null the FK). The
validator also checks each enum against persisted data — narrowing to a
value set that doesn't cover existing rows fails with `VALIDATION` and the
offending value in the message.

### Invoke it

**CLI:**

```sh
canon actions invoke catalog.refine_object_type --input @deal-refine.json
```

**REST:**

```sh
curl -X POST https://api.canonify.app/v1/actions/catalog.refine_object_type \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "input": { "name": "deal", "description": "A sales opportunity against a customer.", "properties": { "stage": { "kind": "mapped", "column": "stage", "type": "enum", "values": ["open", "proposal_sent", "won", "lost"] } }, "links": { "customer": { "target": "customer", "cardinality": "one", "via": "customer_id" } } } }'
```

**MCP** (`actions.invoke` with `name: "catalog.refine_object_type"` and the
same object under `args`).

Once links exist you can navigate them typed:

```sh
canon objects navigate customer cust_… deals   # → the deal rows for one customer
```

Full refinement spec — surfacing hidden columns, opting out of default
CRUD verbs, derived properties — in
[objecttype-refinement.md](resources/objecttype-refinement.md).

---

## Chapter 3 — Bind a StateMachine (`state_machines.register`)

A StateMachine binds to an ObjectType **enum property** and governs which
transitions are legal. Registering it is what gives your transition
actions (Chapter 4) a free guard, a free post-transition state write, and
free `on_enter` / `on_exit` follow-up hooks.

**Action:** `state_machines.register`
**Input shape:** `{ kind: 'state_machine', name, objectType, property, initial, states, transitions, on_enter?, on_exit? }`
**Envelope on success:** `{ data: { name, registered: true }, audit_event_id, approval_status: 'not_required' }`

Bind a machine to the `deal.stage` enum you narrowed in Chapter 2:

```yaml
action: state_machines.register
input:
  kind: state_machine
  name: Deal.stage
  objectType: deal
  property: stage
  initial: open
  states:
    open:           { label: Open }
    proposal_sent:  { label: Proposal sent }
    won:            { label: Won, terminal: true }
    lost:           { label: Lost, terminal: true }
  transitions:
    - { via: deal.send_proposal, from: [open],                to: proposal_sent }
    - { via: deal.win,           from: [proposal_sent],       to: won }
    - { via: deal.lose,          from: [open, proposal_sent], to: lost }
  on_enter:
    won:
      - action: email.send_template
        inputs:
          template:    deal_won
          to:          $row.customer_id
```

Each transition's `via` names the Action that drives it — those actions
get declared in Chapter 4. Once registered, any action whose declaration
carries a matching `transition` marker gets three slabs of machinery
wrapped around its handler **for free**:

1. **Pre-handler guard** — rejects with `INVALID_STATE` if the row's
   current state isn't in `from`.
2. **Post-handler state write** — the runtime auto-writes
   `UPDATE deal SET stage=<to> WHERE id=…` after the handler returns. Your
   handler must **not** write the state column itself (it will be rejected
   with `VALIDATION/direct_state_write`).
3. **Hook enqueue** — `on_enter[<to>]` / `on_exit[<from>]` hooks resolve
   against the post-transition row and enqueue follow-up actions into the
   outbox. The `$row.*` refs in hook `inputs` are passthrough — they
   resolve at execute time against the actual row.

### Invoke it

```sh
canon actions invoke state_machines.register --input @deal-stage.json
```

```sh
curl -X POST https://api.canonify.app/v1/actions/state_machines.register \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d @deal-stage-rest.json   # { "input": { ...the state_machine body... } }
```

MCP: `actions.invoke` with `name: "state_machines.register"`, the body
under `args`. Inspect the result:

```sh
canon state-machines list --json
canon state-machines render Deal.stage   # ASCII state diagram
```

Full cross-validation rules — declaration order, terminal states, `from:
"*"` wildcards — in
[state-machines-binding.md](resources/state-machines-binding.md).

---

## Chapter 4 — Declare a custom ActionType (`actions.declare`)

This is the centerpiece. You declare the full ActionType — name, input
schema, governance, idempotency, optional `transition` marker, and the
**handler** (a JSON document in the handler DSL) — and the runtime
registers it. It appears on every surface immediately; no restart. CLI
per-action sugar (`canon deal.win --deal-id dl_…`) is generated from the
input schema automatically.

**Action:** `actions.declare`
**Input shape:** `{ name, description, object_type, input, governance, idempotency, transition?, handler, fixture? }`
**Envelope on success:** `{ data: { name, registered: true }, audit_event_id, approval_status: 'not_required' }`

The handler is a **linear sequence of typed CRUD ops over ObjectTypes**,
glued by a reference grammar:

| Ref | Meaning |
|---|---|
| `$input.<field>` | A field on the action's schema-decoded input |
| `$<step>.<field>` | A field bound by a prior `read` / `insert` step (via `as`) |
| `$now` | Runtime clock — ISO 8601 UTC |
| `$principal.id` / `.org_id` / `.kind` | Caller identity |

Step kinds: `read` (binds, throws `NOT_FOUND` if missing), `insert`
(optional bind; executor stamps `id` + `created_at` + `updated_at`),
`update`, `delete`. No branching, no loops, no expressions inside values,
no raw SQL — that machinery lives in the surrounding runtime frame, so the
handler is just the domain-side effect.

### A simple transition action (`deal.win`)

When the action's job is a state flip plus one timestamp, the handler is a
single `update`. The state column write is auto-injected by the FSM — never
write it yourself.

```json
{
  "name": "deal.win",
  "description": "Mark a deal as won.",
  "object_type": "deal",
  "input": { "deal_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Deal.stage",
    "from": ["proposal_sent"],
    "to": "won"
  },
  "handler": {
    "steps": [
      { "op": "update", "object": "deal", "where": { "id": "$input.deal_id" }, "set": { "amount": "$input.deal_id" } }
    ],
    "returns": { "deal_id": "$input.deal_id" }
  }
}
```

### A multi-table atomic action (`deal.win_and_activate`)

The richer pattern: read a row, write across two ObjectTypes, return ids —
all inside one transaction. Here, winning a deal also flips the customer to
`active`.

```json
{
  "name": "deal.win_and_activate",
  "description": "Win a deal and activate its customer in one atomic step.",
  "object_type": "deal",
  "input": { "deal_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent"]
  },
  "idempotency": "per-key",
  "handler": {
    "steps": [
      { "op": "read",   "object": "deal",     "where": { "id": "$input.deal_id" }, "as": "deal" },
      { "op": "update", "object": "deal",     "where": { "id": "$deal.id" },       "set": { "amount": "$deal.amount" } },
      { "op": "update", "object": "customer", "where": { "id": "$deal.customer_id" }, "set": { "status": "active" } }
    ],
    "returns": {
      "deal_id": "$deal.id",
      "customer_id": "$deal.customer_id"
    }
  }
}
```

What runs at invocation, inside one tx: idempotency check → governance
check → schema-decode input → (transition guard, if a `transition` marker
is present) → handler steps in order → (auto state-write + hooks, if a
transition) → outbox commit → tx commit → audit row → idempotency cache.

### Invoke it

```sh
canon actions invoke actions.declare --input @deal-win.json
```

```sh
curl -X POST https://api.canonify.app/v1/actions/actions.declare \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "input": { /* the full ActionType declaration above */ } }'
```

MCP: `actions.invoke` with `name: "actions.declare"`, the declaration
under `args`. After it registers, run your new action like any other:

```sh
canon deal.win --deal-id dl_…          # per-action sugar
canon actions describe deal.win --json # the full declaration back
```

`actions.declare` is `idempotency: 'per-key'` — re-submitting byte-identical
JSON is a free replay or a no-op success; submitting **different** JSON
under the same name fails with `INVALID_STATE/declaration_changed` (use
`actions.update` for deliberate changes). Full field-by-field contract,
the static type-checker's rejections, and copy-paste handler bodies in
[actions-multi-step.md](resources/actions-multi-step.md).

---

## Chapter 5 — Compose a ViewSpec (`view_specs.declare`)

The final chapter shapes how the app **renders**. A ViewSpec shadows the
renderer's auto-default for an ObjectType. (ViewSpecs are rendered by the
Integrated-tier web app; declaring one is the same on every surface.) The
declaration is validated against the live registry at declare time —
unknown ObjectTypes, source mismatches, and bad column refs are rejected
before anything persists.

**Action:** `view_specs.declare`
**Input shape:** `{ name: string, object_type: string, spec: ViewSpec }`
**Envelope on success:** `{ data: { name, registered: true }, approval_status: 'not_required', audit_event_id }`

The `spec` is a discriminated union on `type`. The palette is a closed set
of nine variants:

| `type` | Renders |
|---|---|
| `table` | A row-per-record list (or row-per-group, with an aggregate source) |
| `detail` | One record's property grid (with an optional `header` band) |
| `panel` | A form that hosts an ActionType (mutations) |
| `button` | A single action trigger |
| `composite` | A container of nested specs — `layout` selects `stack` (default), `grid`, or `tabs` |
| `metric` | One scalar KPI tile (an aggregate without `groupBy`) |
| `file` | An image / attachment surface |
| `board` | A kanban board grouping records by an enum property |
| `queue` | A prioritized, keyboard-first action worklist |

Start with a list view over `customer`. `name` must be dot-separated
segments (e.g. `Customer.list`); `spec.source.objectType` must agree with
the declared `object_type`; every column ref must resolve to a property or
link on that ObjectType.

```json
{
  "name": "Customer.list",
  "object_type": "customer",
  "spec": {
    "type": "table",
    "source": { "objectType": "customer" },
    "columns": ["name", "email", "status", "deals.count"]
  }
}
```

`deals.count` is a link aggregate — `count` is the only property-less
aggregate; `sum`/`avg`/`min`/`max` take a numeric `:property` argument
(e.g. `deals.sum:amount`). A scalar KPI tile uses the same aggregate shape
without a `groupBy`:

```json
{
  "name": "Customer.active_count",
  "object_type": "customer",
  "spec": {
    "type": "metric",
    "label": "Active customers",
    "source": { "objectType": "customer", "aggregate": { "op": "count" }, "filter": { "status": "active" } }
  }
}
```

### Invoke it

```sh
canon actions invoke view_specs.declare --input @customer-list.json
```

```sh
curl -X POST https://api.canonify.app/v1/actions/view_specs.declare \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "input": { "name": "Customer.list", "object_type": "customer", "spec": { "type": "table", "source": { "objectType": "customer" }, "columns": ["name", "email", "status"] } } }'
```

MCP: `actions.invoke` with `name: "view_specs.declare"`, the body under
`args`. Re-declaring under the same `name` is the deliberate override —
the spec is replaced and re-validated against today's schema. Full
component reference, source filters, and `showWhen` conditioning in
[viewspec-quick-start.md](resources/viewspec-quick-start.md).

### Don't start from a bare table — start from a template

A first-draft view that *renders* is rarely the same as one that *reads well*:
the elevated treatment (a `detail` `header`, `valueColors` status chips, filter
`presets`, a promoted rollup metric, curated columns) lives in knobs you have to
know exist. So don't hand-roll from `{ type: "table", columns: [...] }` — **copy
a starter template** and adapt the ObjectType/property names. The
[starter-templates.md](resources/starter-templates.md) resource ships four
ready-to-declare, well-commented views — an elegant record detail, a scannable
list, a board, and a metric tile — plus a **"reach elegance in 7 moves"**
checklist. Each is current vocabulary and previews/declares as-is.

---

## You shipped an app

Five verbs, in order, and you have a typed, governed, FSM-driven CRM with
custom rendering — all declared, no code. Recap:

1. `schema.propose_migration` — three tables + auto-CRUD.
2. `catalog.refine_object_type` — enums + links + descriptions.
3. `state_machines.register` — `Deal.stage` lifecycle.
4. `actions.declare` — `deal.win`, `deal.win_and_activate`.
5. `view_specs.declare` — `Customer.list`, the active-customers metric.

Verify your work the way the platform does — every invocation wrote an
`audit_event_id`:

```sh
canon audit --since 1d                 # everything you just did
canon audit <audit_event_id>           # one call, full envelope
canon actions list --json              # your new actions are in the catalog
```

## See it: verify your UI work in the app

The CLI confirms the *data* is right. It does **not** confirm the app *looks*
right. **When you build or change a canon, verify your UI work the same way you
verify CLI results** — open the app and confirm the rendered views, the
"Browse data" sidebar (scoping + `part_of` nesting), and the nav match what you
declared. A view that lists the wrong columns, a Browse-data list that surfaces
every table, or a compound whose parts don't nest are all things only a *look*
catches.

You can do this yourself with the **same api-key you use for the CLI** — an
agent gets a short-lived *web* session by exchanging its key, then drives
`app.canonify.app` with the **`agent-browser`** CLI. Your key authenticates the
`/v1` API; it does **not** authenticate the web session directly, so you trade
it for a short-lived session first (the long-lived key never enters the
browser):

```sh
# 1. Exchange your org-scoped agent key for a short-lived web session.
#    (Origin must be an allowed Canonify web origin.)
SESSION=$(curl -s -X POST https://api.canonify.app/api/auth/agent/exchange \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Origin: https://app.canonify.app" | jq -r .token)

# 2. Drive the app as the agent: inject the SESSION (not the key) and open it.
agent-browser open https://app.canonify.app
printf "localStorage.setItem('canonify_token', %s); 'ok'" "\"$SESSION\"" \
  | agent-browser eval --stdin
agent-browser open https://app.canonify.app          # now authenticated
agent-browser snapshot -i                            # read the rendered shell

# 3. Navigate to YOUR app + view and confirm it renders as declared.
agent-browser open https://app.canonify.app/app/<your-app-slug>
agent-browser screenshot /tmp/my-app.png             # eyeball it
```

You'll appear in the app with a clear **"Agent" badge** (you're a first-class
principal, never impersonating a human; the org switcher and account settings
are hidden — your session is scoped to the key's one org). The session is
short-lived; re-exchange when it expires, and revoke the key any time with
`canon key revoke <id>`.

Start the agent-browser workflow with `agent-browser skills get core --full`.

## Group & scope it: the App record

An **App** is a named, ordered grouping of your declared Views — the front door
a user lands on. It owns nothing; each `nav` entry references a View by name.
Declare one with `apps.declare`:

```sh
canon actions invoke apps.declare --input '{
  "slug": "crm",
  "name": "CRM",
  "icon": "briefcase",
  "nav": [
    { "section": "Pipeline", "view": "Deal.pipeline" },
    { "section": "Pipeline", "view": "Customer.list" }
  ],
  "defaultView": "Deal.pipeline"
}'
```

Three optional levers shape the app shell:

- **`defaultView`** — the nav view the app OPENS on, instead of the tile
  overview. Must name one of the app's own `nav` views (an out-of-nav value is
  ignored, never a hard-fail).
- **`browse`** — an explicit list of OBJECT-TYPE names for the app's secondary
  **"Browse data"** sidebar. **Omit it (the default) and the list is DERIVED**
  from the object-types your `nav` views target, each expanded to its whole
  `part_of` compound — so a curated app does NOT dump every org table, and a
  compound's parts nest under their parent automatically. Set `browse` ONLY to
  override that derivation (add a type with no view, or pin an exact set):

  ```sh
  canon actions invoke apps.declare --input '{
    "slug": "crm", "name": "CRM", "nav": [ { "view": "Deal.pipeline" } ],
    "browse": ["deal", "customer", "contract"]
  }'
  ```

  `browse` is a soft lever: names that don't resolve to a real object-type are
  dropped at render, and re-declaring WITHOUT `browse` clears a prior override.

- **`count`** — a per-nav-item live badge. Set `count` on a `nav` entry to
  `{ "objectType": "<type>", "filter?": { … } }` and the item renders a badge
  with the row count of that object-type (optionally filtered) — e.g. a
  "Support" item badged with open tickets. The op is always a row count.
  Absent → no badge.

  ```sh
  canon actions invoke apps.declare --input '{
    "slug": "ops", "name": "Ops",
    "nav": [ { "view": "Ticket.queue", "count": { "objectType": "ticket", "filter": { "status": "open" } } } ]
  }'
  ```

## Crystallize it: the manifest lifecycle (export · version · evolve · apply)

Everything you declared above lives in the org's **live catalog**. To make it a
durable, shareable, version-controlled artifact, **download it as a manifest**:

```sh
canon catalog export ./my-app-canon          # the whole canon → a bundle dir
```

That bundle is a directory of declarative JSON — one file per artifact
(`object-types/`, `actions/`, `state-machines/`, `views/`, `apps/`, `policies/`,
`forms/`, `schema/`, plus a `manifest.json` index). **It is the single
comprehensive source of truth.** You never hand-write a manifest from scratch —
you grow a canon with the five verbs and *export* it. Commit it to git: clean
per-artifact diffs, reviewable in a PR.

Apply it to any org — a teammate's, staging, production — idempotently:

```sh
canon catalog apply ./my-app-canon           # reconcile a canon to the bundle
```

`apply` is **not a blind write**: it replays every declaration back through the
same validating handlers (unknown column, illegal transition, missing
title/icon, decode errors all still fail). **You cannot apply an invalid
manifest.** Re-applying the same bundle is a no-op.

### `apply` is a TOTAL validator — validate first (`canon catalog validate`)

`apply` is the closest thing this platform has to a **compiler**: because you
never write code, the apply-time gate is the only type system you get. It is
**total** — a bundle it accepts renders and runs with no definition-level runtime
error (**applies-green ⟺ works-at-runtime**). Beyond the per-artifact decode above,
apply also cross-checks the linkages between artifacts:

- **Closed shapes — unknown fields are REJECTED, not ignored.** Every artifact
  spec decodes through a CLOSED schema (`onExcessProperty: 'error'`): a key the
  shape doesn't declare fails decode with `invalid_artifact`. Author to the exact
  documented keys, not to intuition. The common traps:
  - A table sorts via **`source.sort`** (a property name; prefix `-` for
    descending, e.g. `"-created_at"`) — there is NO top-level `sort` and NO
    `limit` on a table block.
  - A block's section heading is **`title`**, not `label`. (`label` exists only
    as a *metric*-tile caption, a *button* control caption, or a column-ref
    display hint `{ ref, label }` — never as a table/detail/composite heading.)
  - An App `nav` entry is exactly **`{ section?, view, count? }`** — an `icon` or `label`
    ON a nav entry is rejected. Icons live on the **App** (`{ slug, name, icon? }`)
    or on a **View** (the ViewSpec's own `icon`, validated against the Lucide
    name set), NOT on the nav entry.
  - A `composite` scopes a child list to the parent row through the child's own
    **`source.filter`** (e.g. `{ customer_id: "$context.id" }`), not a field on
    the composite — `composite` carries only `{ layout?, children[], title?,
    showWhen? }`.
- **View → action subject.** Every table `rowActions` / `queue` / `board` action a
  view binds must have a resolvable **subject** — the input param that carries the
  row id. The catalog derives it from the `<object_type>_id` convention; when your
  param doesn't follow it, declare it explicitly on the ActionType with
  `subject: { param, object_type }` (see the base `canonify` skill's *"The
  subject"* section). If neither resolves, apply **fails at apply** naming the
  action + the expected param — instead of applying green and then throwing
  "Argument validation failed" when a user clicks the button.
- **Queue view refs.** A `queue` view's `source.objectType`, `order[].field`,
  `urgency.from` (must be an **enum** property), `row` `$field` / `{{age <field>}}`
  tokens, and `row.peek` (must name a real `detail` view on the same object-type)
  must all resolve — a typo fails the apply, not renders `—` at runtime.
- **Every other view's refs.** `table.columns`, `detail.properties`,
  `board.groupBy`/`columns`, `source.filter`/`sort` keys, and panel/button/board
  action bindings are resolved against the object-type/actions too — a typo'd
  column (`{type:'table', columns:['lable']}`) fails apply. **`validate` now
  replays this same resolution** (tick `6cu`), so a column typo fails `validate`
  too, not just `apply`.

Run that **same total gate without applying** — ideal as a CI pre-flight before you
push a bundle at prod:

```sh
canon catalog validate ./my-app-canon                 # validate against the ambient canon
canon catalog validate ./my-app-canon --live prod     # validate the NEXT version against a deployed canon
```

It is **read-only** (no writes, no reconcile) and reports **every** problem in one
pass (apply stops at the first), each naming the artifact + cause, and **exits
non-zero** when the bundle is invalid so CI can gate on it. The diagnostics carry a
stable `code` you can scan for:

| `code` | Means |
|---|---|
| `invalid_artifact` | An artifact fails its declaration schema (missing `governance` or `idempotency`, malformed `handler`, …) — apply is not more permissive than `declare`. |
| `unresolvable_subject` | A view row/queue action has no resolvable subject (no `<type>_id` param and no explicit `subject`, or a `subject` naming a param that isn't a real input). |
| `unresolvable_ref` | A queue `source.objectType` / `order.field` / `urgency.from` / `$field` token / `row.peek` doesn't resolve. |
| `unknown_column_ref` (& `source_object_type_mismatch`, `invalid_board_group_by`, `unknown_action`, `invalid_column_format`, …) | A table/detail/board/panel/button view's column / property / groupBy / filter / sort / action ref doesn't resolve — the declaration-layer checks `validate` now replays (tick `6cu`). |

**apply-rejects ≡ validate-rejects** — a clean `canon catalog validate` means
`apply` will accept the bundle, **including for a greenfield bundle**. The CLI
`canon catalog validate` runs a **faithful local dry-run**: it boots a throwaway
in-memory canon, materializes your bundle's schema + object-types for real, then
runs every check against them — so a typo'd column or a wrong-shape `rowActions`
over a brand-new object-type is caught here, not just at apply. Use it as your CI
pre-flight. (The REST/MCP `catalog.validate` *action* is a lighter static check
that still defers object-type-dependent checks for same-bundle-new types — the CLI
is the faithful one; `apply` is always the backstop.)

### apply is fully declarative — it prunes (ticket 95j)

`apply` reconciles the live **`managed` partition** to **exactly** what the
bundle declares. It is **not** additive: a managed artifact that was present in
the org but has been **dropped from the bundle is DELETED**
(object-type refinements, actions, state machines, views, policies, forms, apps).
So the bundle is the whole truth for the managed partition — remove an artifact's
file, re-apply, and it's gone. The deletion runs in reverse-dependency order
(apps → forms → policies → views → state_machines → actions → object_types).

**`tenant`-partition artifacts are NEVER pruned.** Every prune is gated on
`managed_by = 'managed'`; anything a customer grew organically in the org
survives an apply that doesn't mention it. (The managed/tenant split is the
app-dev provenance seam — `apply` only ever owns the managed half.)

> Atomicity is **per-substep**, not whole-apply: a mid-apply failure can leave a
> partial state (the steps that already committed stay committed). That is safe
> because every step is upsert / reconcile-in-place — a clean re-apply converges.

### Optimistic concurrency — `--expected-revision` (ticket 63b)

The exported manifest carries a `revision` field — a monotonic integer the canon
bumps by 1 on each successful apply (a **fresh canon is `0`**). `catalog export`
prints it; read it off `manifest.revision`. Pass it back to guard against a
lost update when two authors target the same canon:

```sh
canon catalog apply ./my-app-canon --expected-revision 7
```

If the canon's current revision no longer equals what you pinned, apply is
**rejected before any write** with a `STALE_REVISION` 409 (`current` / `expected`
in the details) — re-export, rebuild your desired state, re-apply. Omit the flag
and apply proceeds unconditionally. A concurrent apply already in flight is a
distinct signal — `APPLY_IN_PROGRESS` (retry shortly), not stale.

### Two ways to evolve the manifest (pick by the change)

- **Organic → re-export.** Declare more through the five verbs (per-step
  validation), then `canon catalog export` again. Best when exploring.
- **Edit the bundle → apply.** The bundle is plain declarative IaC — edit
  `my-app-canon/**/*.json` by hand or script, then `canon catalog apply`. Best
  for bulk / mechanical / cross-cutting changes (retitle everything, swap all
  view icons in one pass). `apply` re-validates, so a sloppy edit fails the
  apply rather than corrupting the org.

### Author without a live org — `catalog scratch` (ticket erl)

Sometimes you want to grow a bundle organically (per-step validation) but have
**no live org to touch** — drafting, CI, or a throwaway exploration. `catalog
scratch` boots a fresh **empty in-process Canon**, replays a list of invocations
against it, exports the resulting bundle to a directory, and tears the Canon
down:

```sh
canon catalog scratch ./invocations.json ./my-app-canon
```

The invocations file is `{ "invocations": [ { "action": "...", "input": {...} }, … ] }`
— the same five verbs you'd run live, in order. A failing invocation aborts with
a message naming the action; on success you get the exported bundle and nothing
durable is left behind (no live org, no pollution). Use it for ephemeral
authoring; use `export` + `apply` when you're working against a real org.

### The guarantee that makes the manifest trustworthy

Author → `export` → `apply` to a fresh org → **byte-identical**. Anything you can
declare survives the round-trip (it's a conformance gate). So the committed
bundle faithfully reconstructs the app — that's why it's safe to treat as *the*
source of truth. Keep the committed bundle and the live org in sync: export +
commit after changes; never let live drift from the manifest.

## Three surfaces, one runtime (isomorphism)

This skill itself is served identically on every surface — that is the
same guarantee your declared actions get. Read it back over any of:

```sh
canon skill --name canonify-app-authoring instructions   # this body
canon skill --name canonify-app-authoring resources      # the chapter list
canon skill --name canonify-app-authoring resource resources/actions-multi-step.md
canon skills list                                         # this + the base canonify skill
```

Over REST the same content is at
`GET /v1/skills/canonify-app-authoring`,
`GET /v1/skills/canonify-app-authoring/instructions`,
`GET /v1/skills/canonify-app-authoring/resources`, and
`GET /v1/skills/canonify-app-authoring/resources/<path>`. (These skill
routes are public — discovery before auth.) Over MCP you discover the live
catalog with the `actions.list` tool and invoke every declaration verb in
this tutorial through the `actions.invoke` tool — `{ name, args }` — so the
whole authoring loop runs over MCP just as it does over CLI and REST.

## Resources

- [resources/schema-bootstrap.md](resources/schema-bootstrap.md) — Chapter 1, `schema.propose_migration`
- [resources/objecttype-refinement.md](resources/objecttype-refinement.md) — Chapter 2, `catalog.refine_object_type`
- [resources/state-machines-binding.md](resources/state-machines-binding.md) — Chapter 3, `state_machines.register`
- [resources/actions-multi-step.md](resources/actions-multi-step.md) — Chapter 4, `actions.declare`
- [resources/viewspec-quick-start.md](resources/viewspec-quick-start.md) — Chapter 5, `view_specs.declare`
- [resources/starter-templates.md](resources/starter-templates.md) — copy-paste elevated ViewSpec templates + the "reach elegance in 7 moves" checklist

## See also

- [`../canonify/SKILL.md`](../canonify/SKILL.md) — the base reference manual (each verb in isolation, the runtime frame, error codes)
- [`../canonify/resources/handler-dsl-cookbook.md`](../canonify/resources/handler-dsl-cookbook.md) — more handler bodies + the anti-patterns the type-checker rejects
- [`../../../AGENTS.md`](../../../AGENTS.md) — the Canon CLI cheat-sheet and contributor conventions
