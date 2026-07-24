---
name: canonify
description: How to operate Canonify declaratively — the manual served by `canon skill instructions` and `GET /v1/skill/instructions`.
version: 1.0.0
---

# Canonify — the manual

Canonify is a **declarative platform**. You build apps by invoking
Actions that declare schema, ObjectTypes, StateMachines, and custom
ActionTypes. **You never write code.** Every artefact in the catalog —
tables, typed object shapes, state machines, multi-step transition
handlers — is the result of an Action you invoked through CLI, REST,
or MCP. The runtime parses the declarations, runs them inside a single
transactional frame with audit/outbox/idempotency/governance baked in,
and surfaces the result as a uniform envelope on every surface.

The platform vocabulary is small. Read this manual top to bottom once.
Then bookmark the cookbook ([resources/handler-dsl-cookbook.md](resources/handler-dsl-cookbook.md))
and the error-code reference ([resources/error-codes.md](resources/error-codes.md))
for in-flight reference.

If this is your first invocation, run:

```sh
canon actions list --json
canon objects list-types
canon schema describe
```

— that's the live catalog. Everything below is how to add to it.

### Which tier is this? (Integrated)

Canonify apps come in three tiers, split on **who renders the UI and who
Canon serves it to**:

| Tier | UI | Reach | Surface |
|---|---|---|---|
| **Integrated** (Tier 1) | Canon renders ViewSpec in `apps/web` | internal org principals only | declarative — ViewSpec + Actions |
| **Hosted** (Tier 2) | Canon serves *your* static assets | internal *or* external | static HTML/CSS/JS + Canon API |
| **External** (Tier 3) | *you* host the app | external | generated REST + SDK |

**These skills cover the Integrated tier** — declarative authoring of
schema, ObjectTypes, StateMachines, Actions, and ViewSpec rendered by
`apps/web`. Hosted/External authoring is out of scope here until a real
story pulls those tiers. See the glossary
([[app-tiers]] / [`repo-wiki/glossary/glossary.md`](../../../repo-wiki/glossary/glossary.md))
for the full definitions.

### Pull the deep-dive skills

This manual is the base skill. The companion deep-dives
(`canonify-app-authoring`, `canonify-objects-actions`,
`canonify-viewspecs`, `canonify-scenarios`, `canonify-isomorphism`,
and `lore` — how to OPERATE on a Canon as an agent: discovery,
invocation, approval flows, proposal-authoring, knowledge citation,
and error handling) ship in the same `@canonify/skills` package.
Vendor them into your agent workspace with:

```sh
canon skills                                  # list every published skill
canon skills install canonify-app-authoring   # vendor one skill
canon skills sync                             # refresh every skill at once
```

Both write to **`.claude/skills/`** and **`.agents/skills/`** — the two places an
agent reads skills from — so one command keeps every agent in the repo current.
Pass `--target <dir>` to write a single directory of your choosing instead.

No `canon` binary? The same install works the conventional way:
`npx @canonify/skills install canonify-app-authoring`.

---

## The four declaration verbs

Every change to the catalog goes through one of four platform Actions.
Memorise this table — the rest of the manual is a deep-dive on each.

| Action | Use it to | Reference |
|---|---|---|
| `schema.propose_migration` | Create or alter tables. Auto-CRUD synthesises a default ObjectType + the five CRUD ActionTypes from the new table. | [resources/declaring-tables.md](resources/declaring-tables.md) |
| `catalog.refine_object_type` | Narrow string columns to typed enums, surface hidden columns, add typed links between ObjectTypes (incl. `on_delete` cascade), set human-readable descriptions, opt out of default CRUD verbs. | [resources/declaring-objecttypes.md](resources/declaring-objecttypes.md) |
| `state_machines.register` | Bind a Phase 18.5 finite-state machine to an ObjectType enum property. Each transition names the Action that drives it. `on_enter`/`on_exit` hooks enqueue follow-up actions. | [resources/declaring-state-machines.md](resources/declaring-state-machines.md) |
| `actions.declare` | Register a custom multi-step ActionType. The handler is a JSON document in the **handler DSL** — typed CRUD steps over ObjectTypes glued by the `$input` / `$<step>` / `$now` / `$principal` ref grammar, executed inside the runtime's tx + audit + outbox + FSM frame. | [resources/declaring-actions.md](resources/declaring-actions.md) |

The order in a fresh org: tables → ObjectType refinements →
StateMachines → custom ActionTypes → run the business loop.
[`tests/scenarios/minimal-crm/session.yaml`](../../../tests/scenarios/minimal-crm/session.yaml)
is a worked end-to-end session demonstrating all four verbs.

---

## Phase 1 — Tables (`schema.propose_migration`)

A table is one Action call. Auto-CRUD generates the rest.

```yaml
action: schema.propose_migration
input:
  description: lead table — pre-conversion sales prospect
  sql: |
    CREATE TABLE lead (
      id            TEXT PRIMARY KEY,
      name          TEXT NOT NULL,
      email         TEXT NOT NULL,
      source        TEXT NOT NULL,
      status        TEXT NOT NULL DEFAULT 'new',
      contacted_at  TEXT,
      qualified_at  TEXT,
      converted_at  TEXT,
      customer_id   TEXT,
      created_at    TEXT NOT NULL,
      updated_at    TEXT NOT NULL
    )
```

Conventions (from `AGENTS.md`):

- `id TEXT PRIMARY KEY` — prefixed ULIDs (`ld_…`, `cust_…`).
- `created_at` / `updated_at` are `TEXT NOT NULL` ISO 8601 UTC.
  Never `INTEGER` epoch.
- System tables are `_<name>`; that namespace is platform-owned. Don't
  declare `_<your_table>`.

After the migration commits, the new actions are available
**immediately** on every surface:

```sh
canon actions list --json | jq '.[].name' | grep '^lead\.'
# lead.create
# lead.read
# lead.update
# lead.delete
# lead.list
```

See [resources/declaring-tables.md](resources/declaring-tables.md) for
the full contract — what auto-CRUD generates, how to override defaults,
and failure modes.

---

## Phase 2 — ObjectTypes (`catalog.refine_object_type`)

Auto-CRUD's default ObjectType is permissive — every column is a
`mapped` `string`, no links, no description. Refine it to declare the
**rich** domain shape.

The load-bearing pattern is enum narrowing — a state column must be a
typed `enum` before a StateMachine can bind to it:

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

The validator checks the enum against persisted data — narrowing to a
value set that doesn't cover existing rows fails with `VALIDATION` and
the offending value in the message.

Links enable typed navigation:

```sh
canon objects navigate customer cus_… contacts   # → list of contact rows
```

Cascade-on-link controls what happens when a parent is deleted:

```yaml
links:
  contacts: { target: contact, cardinality: many, via: customer_id, on_delete: cascade }
  deals:    { target: deal,    cardinality: many, via: customer_id, on_delete: restrict }
```

Policies: `restrict` (default — `INVALID_STATE` if children exist),
`cascade` (delete children first), `set_null` (null-out the FK).
Honoured by **every** delete path, not just declared handlers.

See [resources/declaring-objecttypes.md](resources/declaring-objecttypes.md)
for the full refinement spec, including how to surface platform-hidden
columns (the precise prerequisite for `lead.convert` writing through
`org_id`).

---

## Phase 3 — StateMachines (`state_machines.register`)

State machines bind to an ObjectType enum property and govern legal
transitions:

```yaml
action: state_machines.register
input:
  kind: state_machine
  track: customer
  name: Lead.status
  objectType: lead
  property: status
  initial: new
  states:
    new:          { label: New }
    contacted:    { label: Contacted }
    qualified:    { label: Qualified }
    converted:    { label: Converted, terminal: true }
    disqualified: { label: Disqualified, terminal: true }
  transitions:
    - { via: lead.contact,    from: [new],                       to: contacted }
    - { via: lead.qualify,    from: [contacted],                 to: qualified }
    - { via: lead.disqualify, from: [new, contacted, qualified], to: disqualified }
    - { via: lead.convert,    from: [qualified],                 to: converted }
  on_enter:
    converted:
      - action: email.send_template
        inputs:
          template:    welcome
          to:          $row.email
          customer_id: $row.customer_id
```

Once registered, transition Actions whose declaration carries a
matching `transition` marker get three slabs of machinery wrapped
around their handler **for free**:

1. **Pre-handler guard** — rejects with `INVALID_STATE` if the row's
   current state isn't in `from`.
2. **Post-handler state write** — runtime auto-writes
   `UPDATE <table> SET <property>=<to> WHERE id=…` after the handler
   returns. The handler must NOT write the state column itself
   (`actions.declare` rejects with `VALIDATION/direct_state_write`).
3. **Hook enqueue** — `on_enter[<to>]` / `on_exit[<from>]` hooks
   resolve against the post-transition row and enqueue into `_outbox`.

The `$row.*` / `$row_before.*` refs in hook `inputs` are
**passthrough** — they resolve at execute time against the actual row,
not against session scope.

See [resources/declaring-state-machines.md](resources/declaring-state-machines.md)
and [Phase 18.5](../../phases/18.5-state-machines.md) for the
cross-validation rules.

---

## Phase 4 — Custom ActionTypes (`actions.declare`)

This is the centerpiece. An agent declares the full ActionType — name,
input schema, governance, idempotency, transition marker, handler
(the DSL document), fixture — and the runtime registers it. Every
surface picks it up immediately; no restart.

The handler is a **linear sequence of typed CRUD ops over
ObjectTypes**, glued by a reference grammar:

| Ref | Meaning |
|---|---|
| `$input.<field>` | Field on the action's schema-decoded input |
| `$<step>.<field>` | Field on a previously bound `read`/`insert` step result |
| `$now` | Runtime clock (ISO 8601 UTC) |
| `$principal.id` / `.org_id` / `.kind` | Caller identity |

Step kinds: `read` (binds), `insert` (optional bind), `update`,
`delete`. No branching, no loops, no expressions inside values, no raw
SQL — see [Phase 17.5 §1](../../phases/17.5-declarative-handlers.md#1-the-handler-dsl--vocabulary)
for the deliberate omission list.

### The canonical worked example: `lead.convert`

The four-step atomic multi-table write. Same handler in three
representations — agents pick whichever the surface they're driving
prefers.

JSON for `--input @file.json`:

```json
{
  "name": "lead.convert",
  "description": "Convert a qualified lead into a Customer + primary Contact.",
  "object_type": "lead",
  "input": { "lead_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Lead.status",
    "from": ["qualified"],
    "to": "converted"
  },
  "handler": {
    "steps": [
      { "op": "read",   "object": "lead",     "where": { "id": "$input.lead_id" }, "as": "lead" },
      { "op": "insert", "object": "customer", "values": { "name": "$lead.name", "email": "$lead.email", "org_id": "$principal.org_id" }, "as": "customer" },
      { "op": "insert", "object": "contact",  "values": { "customer_id": "$customer.id", "name": "$lead.name", "email": "$lead.email", "role": "primary" }, "as": "contact" },
      { "op": "update", "object": "lead",     "where": { "id": "$input.lead_id" }, "set": { "customer_id": "$customer.id", "converted_at": "$now" } }
    ],
    "returns": {
      "lead_id": "$input.lead_id",
      "customer_id": "$customer.id",
      "contact_id": "$contact.id"
    }
  }
}
```

What runs at invocation time, inside one tx:

1. Idempotency check → governance check → schema-decode input
2. Phase 18.5 transition guard asserts `lead.status === 'qualified'`
3. Executor runs the four steps in order — `read` binds `$lead`,
   `insert customer` stamps `id` (`cus_…`) + `created_at` + `updated_at`
   and binds `$customer`, `insert contact` references `$customer.id`,
   `update lead` stamps `customer_id` and `converted_at`
4. Phase 18.5 post-update writes `lead.status='converted'` (the
   handler did NOT write it — that's the contract)
5. Phase 18.5 on_enter[`converted`] hook enqueues the welcome-email
   row into `_outbox`
6. Audit row written, outbox rows committed, tx commits
7. Envelope: `{ data: { lead_id, customer_id, contact_id },
   audit_event_id, … }`

See [resources/handler-dsl-cookbook.md](resources/handler-dsl-cookbook.md)
for more examples — simple state transitions (`lead.contact`),
insert-only with returns, read-then-update by id, and the anti-patterns
the typechecker will reject.

### Built-in action surfaces you invoke (don't declare)

Not every action is one you author. The platform also ships **built-in**
ActionTypes — registered unconditionally, not via `actions.declare` — that you
**invoke** like any other. The largest is **e-signature**: a native signing
primitive over `signature_request` (single signer), `signature_envelope`
(ordered multi-signer), and `signature` (the immutable signed record), plus a
saved `member_signature`. It spans external signers (who redeem an emailed
`link_token`) and org members (session-authed), with mandatory signing consent,
`link_token` redaction from the audit trail, server-derived signing evidence, and
propagating terminal lifecycles (decline ends an envelope; void clears the
countersign queue; completion is automatic). The full invoke-by-invoke surface is
in the `canonify-objects-actions` deep-dive
([resources/esign-actions.md](../canonify-objects-actions/resources/esign-actions.md));
the browser-portal side is the operations doc
`repo-wiki/operations/signing-protocol.md`.

---

## The runtime frame

Every `runtime.invoke(name, args, principal)` call goes through one
in-process orchestrator. Surfaces never reimplement.

```ts
{
  data,              // typed object | list | rows; null on error
  audit_event_id,    // every invocation writes one (even failed ones)
  approval_status,   // 'not_required' | 'pending' | 'approved' | 'rejected'
  knowledge_used,    // Phase 22 — populated when knowledge fires
  outcome_signals,   // populated by post-call feedback
  error?,            // present only on failure: { code, message, issues, details? }
}
```

The orchestrator stages (from
[`packages/canon/src/runtime.ts`](../../../packages/canon/src/runtime.ts)):

1. **Catalog lookup.** Unknown action name → `NOT_FOUND` envelope.
2. **Idempotency check.** Hit on `(action_name, idempotency_key)` →
   cached envelope replayed verbatim. The cache is keyed per
   declaration; identical retries are free.
3. **Governance check.** Principal kind in `allowed_principals`?
   Approval required and approved? Deny → `FORBIDDEN` envelope +
   audit row with `status='denied'`.
4. **Schema-decode input.** Bad input → `VALIDATION` envelope with the
   issues array in `error.issues`.
5. **BEGIN tx.** Open a transaction on the org's libSQL.
6. **Phase 18.5 transition guard** (if action carries a `transition`
   marker). Reject → `INVALID_STATE` + audit `status='denied'`.
7. **Handler runs.** For declared actions: the DSL executor walks
   steps. For platform-tier TS handlers: the closure runs directly.
   `ActionDomainError` throws are caught and rendered into the
   envelope's `error` field with the appropriate code.
8. **Phase 18.5 post-update.** Auto-write the state column to the
   `transition.to` value.
9. **Phase 18.5 on_enter / on_exit hooks.** Resolve refs against the
   post-transition row; enqueue into `_outbox`.
10. **Outbox rows enqueued** by the handler are committed.
11. **COMMIT tx**.
12. **Audit event written.** `_audit_event` row outside the business
    tx so it survives a rollback. Status: `success`, `denied`, or
    `error`.
13. **Idempotency cache stored.** If `idempotency_key` was supplied,
    the envelope is cached for future replays.

**ActionDomainError** is the typed escape hatch — handlers raise it to
turn a domain failure into a typed envelope rather than an
`INTERNAL_ERROR`. Kinds: `VALIDATION`, `NOT_FOUND`, `FORBIDDEN`,
`INVALID_STATE`. CONFLICT and RATE_LIMITED are reserved for storage
and rate-limiting kinds added in later phases — the envelope shape is
stable. See [resources/error-codes.md](resources/error-codes.md) for
the full table including CLI exit-code mapping.

---

## Idempotency

Three postures, declared per ActionType:

- `none` — every call runs. Use for FSM transition actions where the
  state guard is the natural retry barrier.
- `per-key` — `idempotency_key` from the caller becomes the cache
  key. First call's envelope is cached; retries with the same key
  replay it. Use for destructive operations the caller will retry on
  network failure.
- `natural-key` — runtime hashes the input shape itself. Use when the
  input is the de facto unique key (e.g.,
  `email.send_template { template, to, customer_id }`).

`actions.declare` is `per-key` — re-submitting byte-identical JSON is
either a cached replay or a no-op success. Submitting **different**
JSON under the same name fails with `INVALID_STATE/declaration_changed`;
use `actions.update` for deliberate changes.

---

## Audit, outbox, FSM frame

Three durable side-effects every successful invocation produces:

- **`_audit_event`**: one row per call, written outside the business
  tx. Carries `action`, `principal_id`, `principal_kind`,
  `org_id`, `status`, `input_summary`, `error_code` (on failure),
  `created_at`, and the `audit_event_id` returned in the envelope.
  Survives handler rollback so denied/errored calls are recoverable.
- **`_outbox`**: rows enqueued by the handler (or by the FSM hook
  layer) that represent asynchronous follow-up actions —
  `email.send_template`, webhook dispatches, etc. Enqueued atomically
  with the handler's writes; committed only on tx success. Drained
  by a separate dispatcher process.
- **FSM state column write**: auto-injected by Phase 18.5 when the
  action carries a `transition` marker. The handler **never** writes
  the FSM-bound property directly.

This is why the declarative DSL doesn't need branching: governance,
state guards, post-state-write, and outbox-effect machinery all live
in the surrounding runtime frame. The handler is just the
**domain-side effect** — what changes in the typed data model.

---

## Discovering what's already registered

Before declaring anything, look:

```sh
canon actions list --json                   # every action — platform + custom
canon actions describe lead.convert --json  # one action — full declaration
canon objects list-types                    # every ObjectType
canon schema describe                       # overview
canon schema describe --object lead         # one ObjectType — full shape
canon state-machines list --json
canon state-machines render Lead.status     # ASCII state diagram
```

Every per-action sugar command supports `--explain` as a shortcut:

```sh
canon lead.convert --explain
```

— equivalent to `canon actions describe lead.convert --json`. Use it
when an envelope returned `VALIDATION` and you need the input schema
to figure out what the action wanted. The payload includes a
`cli_example` field — a ready-to-paste `canon <name> --input '<args>'`
invocation derived from the action's fixture args, so you can copy it,
edit the values, and run rather than hand-assembling the input.

### Introspecting your work

Every invocation — success, denial, or error — writes an `_audit_event`
row (Phase 17.5). To see what you've done recently, including denials,
failures, durations, and the principal each call ran as, use:

```sh
canon audit [--since 7d] [--action <name>] [--principal <id>]
canon audit <audit_event_id>          # full untruncated envelope detail
```

The detail view is the recovery path when a prior invocation's
envelope is no longer in scrollback. `canon actions invoke audit ...`
is the meta-tool equivalent if you prefer the uniform action path.

If a server-bound command fails immediately with `canon … is older than
the minimum supported … — run: canon upgrade` (HTTP 426,
`CLI_UPGRADE_REQUIRED`), your binary is below the platform's compatibility
floor — run `canon upgrade` and retry. See
[resources/error-codes.md](resources/error-codes.md#the-cli-version-floor--http-426-cli_upgrade_required)
for the floor policy and the CLI's own preflight nag.

When something fails in a way you can't explain, file a report against
the platform maintainers:

```sh
canon report "<failing-command>" "<error-message>" [--context @logs.txt]
```

The `canon` binary appends a hint pointing at `canon report` to every
non-zero exit so this flow is one copy-paste away. Reports submit the
Canonify-ops org's public bug-report form and print the created report's
id on success — hold onto it, it's your handle for the two lookup
commands below.

**Check a report's status by id** — no login required; the id itself is
the capability (it's unguessable and only ever printed to the person who
filed the report):

```sh
canon report status <id> [--quiet]
```

Prints the report's `id`, `status`, `resolution`, `created_at`,
`command`, `error_message`, `expected`, `severity`, and `source` as JSON
(`--quiet` prints just the `status` value). Exit codes: `0` found, `2`
unknown or malformed id ("report not found"), `1` network/server error.

**List the reports you filed:**

```sh
canon report list [--status <state>] [--output json|table]
```

Requires being logged in (`canon auth login`) — the server matches your
account's *verified email* against each report's `reporter_email`, so
this is identity-based, not a local history file. `--status` filters to
one lifecycle state (`open`, `triaged`, `in_progress`, `closed`,
`canceled`). Prints a table by default (id, status, filed date, error
snippet, resolution); `--output json` prints the raw array. Exit codes:
`0` ok (including an empty result), `2` invalid `--status` value, `4` not
logged in.

**Limitation:** an agent API key with no `user_email` configured has no
verified email to match against, so `canon report list` returns empty
for it even if it filed reports — `canon report status <id>` still works
for any id the agent holds. Set `user_email` in the CLI config (or file
reports from a logged-in user identity) to make them show up in `list`.

See the `## Canon CLI cheat-sheet` section in
[`AGENTS.md`](../../../AGENTS.md) for the full daily-use surface.

---

## Surfaces — three projections, one runtime

The same catalog is rendered to three transports. They must produce
**byte-identical envelopes** for the same call — that's the
conformance gate (Phase 17.6 §10).

| Surface | Discovery | Invocation |
|---|---|---|
| **REST** | `GET /v1/actions`, `GET /v1/objects/_types`, `GET /v1/schema_describe` | `POST /v1/actions/<name>` with `{ input, idempotency_key? }` body; `Authorization: Bearer <api_key>` or `Session <token>`; `X-Principal-Json` for testing. |
| **CLI** | `canon actions list`, `canon objects list-types`, `canon schema describe` | `canon actions invoke <name> --input <json\|@file\|->` OR the per-action sugar `canon <name> --field value`. Output `--output json` for agents, `--output text` for humans. |

The CLI also accepts a **hyphen-alias** form for any dotted action name:
`canon <group> <verb> …` maps to `<group>.<verb>` with hyphens in both
parts rewritten to underscores — so `canon catalog refine-object-type
--input @f.json` dispatches to `catalog.refine_object_type`, and `canon
view-specs declare` → `view_specs.declare`. With `--input` it routes
through `actions invoke`; with flat flags or `--explain` it uses the
per-action sugar path. The dotted form is always valid too; both
dispatch identically.
| **MCP** | `actions.list` tool | `actions.invoke` tool with `{ name, input }`. Same JSON-RPC envelope. |

**Scoping a single invocation to an org — `--org`.** Every `canon`
command accepts a global `--org <org-id-or-slug>` that scopes **just
that invocation** to the named org. It takes an `org_…` id directly, or
a slug/name resolved against your memberships (the same set `canon org
switch` uses). Precedence is `--org > CANONIFY_ORG_ID > config`, and —
unlike `canon use` / `canon org switch` — it **never writes
`~/.canon/config.json`**. Reach for it whenever more than one
agent/terminal shares a config: a concurrent `canon use` elsewhere
can't clobber your `--org`, and yours can't clobber theirs. `--org`
with no value, or a slug matching no membership, exits with a usage
error (code 64) and lists your orgs.

```sh
canon actions invoke customer.create --input @c.json --org acme-prod
canon actions list --org org_01H…     # id form, no membership lookup
```

If a session passes via CLI but fails via REST or MCP, that's a
surface-isomorphism regression — caught automatically in CI. **You
should never need to think about which surface you're on**; the
envelope shape is the same.

---

## A complete end-to-end session

[`tests/scenarios/minimal-crm/session.yaml`](../../../tests/scenarios/minimal-crm/session.yaml)
is the canonical complete session: an agent boots an empty Canon org,
declares four tables, refines five ObjectTypes, registers two
StateMachines (`Lead.status`, `Deal.stage`), declares seven custom
ActionTypes (the seven transition Actions including `lead.convert`),
then runs the full SMB sales loop end-to-end. No TS. Every step is
something an agent could type into `canon`.

Read it. It's the working proof of the platform you're operating.

---

## Resources

- [resources/declaring-tables.md](resources/declaring-tables.md) — `schema.propose_migration`
- [resources/declaring-objecttypes.md](resources/declaring-objecttypes.md) — `catalog.refine_object_type`
- [resources/declaring-state-machines.md](resources/declaring-state-machines.md) — `state_machines.register`
- [resources/declaring-actions.md](resources/declaring-actions.md) — `actions.declare`
- [resources/handler-dsl-cookbook.md](resources/handler-dsl-cookbook.md) — copy-paste handler JSON, anti-patterns the typechecker rejects
- [resources/error-codes.md](resources/error-codes.md) — every `ActionDomainError` kind, when it fires, recovery hint, CLI exit code; also the HTTP-level CLI version floor (426 `CLI_UPGRADE_REQUIRED`)

## Source-of-truth references

- [Phase 17.5 — Declarative Handlers](../../phases/17.5-declarative-handlers.md) — the spec this manual implements
- [Phase 17.6 — CLI Ergonomics](../../phases/17.6-cli-ergonomics.md) — surfaces, the skill REST routes, exit-code contract
- [Phase 18.5 — StateMachines](../../phases/18.5-state-machines.md) — FSM frame
- [`AGENTS.md`](../../../AGENTS.md) — contributor contract, conventions, what NOT to do
- [`packages/canon/src/handler-dsl/schema.ts`](../../../packages/canon/src/handler-dsl/schema.ts) — the effect/Schema your handler JSON parses against
- [`packages/canon/src/runtime.ts`](../../../packages/canon/src/runtime.ts) — the orchestrator
