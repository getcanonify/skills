---
title: Chapter 4 — Declare a custom ActionType
description: Use actions.declare to register a multi-step ActionType whose handler is a JSON DSL — typed CRUD steps over ObjectTypes glued by the $input / $<step> / $now / $principal ref grammar, executed inside the runtime's tx + audit + outbox + FSM frame.
---

# Chapter 4 — Declare a custom ActionType (`actions.declare`)

This is the centerpiece of declarative authoring. You write the full
ActionType as JSON — name, input schema, governance, idempotency, optional
`transition` marker, and the **handler** (a JSON document in the handler
DSL) — and `actions.declare` registers it. It is live on every surface
immediately, with no restart, and CLI per-action sugar
(`canon deal.win --deal-id dl_…`) is generated from the input schema
automatically.

> All action / object names below are illustrative CRM examples.

## The action

| | |
|---|---|
| **Action** | `actions.declare` |
| **Input** | `{ name, description, object_type, input, governance, idempotency, transition?, handler, fixture? }` |
| **Envelope** | `{ data: { name, registered: true }, audit_event_id, approval_status: 'not_required' }` |
| **Idempotency** | `per-key` — identical re-submits are a free replay / no-op |

## The handler DSL — one screen

The handler is a **linear sequence of typed CRUD ops over ObjectTypes**,
glued by a reference grammar. Every JSON handler that decodes against the
DSL schema executes inside the runtime; see the base skill's
[handler-dsl-cookbook.md](../../canonify/resources/handler-dsl-cookbook.md)
for the schema-backed examples and the anti-patterns the type-checker
rejects.

| Ref | Meaning |
|---|---|
| `$input.<field>` | A field on the action's schema-decoded input |
| `$<step>.<field>` | A field bound by a prior `read` / `insert` step (named via `as`) |
| `$now` | Runtime clock — ISO 8601 UTC |
| `$principal.id` / `.org_id` / `.kind` | Caller identity |
| `"literal"`, `42`, `true`, `null` | Plain JSON literals |

Step kinds:

| op | Effect | Binds via `as`? |
|---|---|---|
| `read` | SELECT one row; throws `NOT_FOUND` if missing | required |
| `insert` | INSERT a row; executor stamps `id`, `created_at`, `updated_at` | optional |
| `update` | UPDATE by `where`; executor stamps `updated_at` | — |
| `delete` | DELETE by `where`; respects the ObjectType's `on_delete` links | — |

Deliberately absent: `if`/`else`, loops, expressions inside values, raw
SQL. That machinery lives in the surrounding runtime frame, so a handler is
only the **domain-side effect**.

## Field-by-field (the declaration envelope)

- **`name`** — `<object>.<verb>`, lowercase, dot-separated (regex
  `^[a-z][a-z0-9]*\.[a-z][a-z0-9_]*$`). May not collide with reserved
  meta-tool roots (`actions`, `objects`, `sql_read`, `schema`, `skill`,
  `state-machines`, `auth`, `keys`, `orgs`, `users`, `config`, `upgrade`).
- **`description`** — required, one crisp sentence; shows in
  `canon actions list` and `--explain`.
- **`object_type`** — the registered ObjectType the action operates on.
- **`input`** — a JSON-Schema-ish descriptor. Field types: `string`,
  `number`, `boolean`. Optional: `{ type: 'string', optional: true }`.
  Prefixed-id: `{ type: 'string', prefix: 'dl_' }`.
- **`governance`** — `{ requires_approval, allowed_principals }`;
  principals are a non-empty subset of `user`, `agent`, `service_account`.
- **`idempotency`** — `none` | `per-key` | `natural-key`.
- **`transition`** (optional) — `{ state_machine, from, to }`; marks this
  as an FSM transition action (Chapter 3 wraps it in guard + auto-write +
  hooks).
- **`handler`** — `{ steps[], returns? }`, the DSL document.
- **`fixture`** (optional, recommended) — the conformance-gate fixture so
  the action verifies byte-identically across REST/CLI/MCP.

> **The declaration is closed.** `actions.declare` rejects any field not in the
> list above (`onExcessProperty: 'error'`). In particular the platform-internal
> ActionType markers below are **NOT** declarable through `actions.declare` —
> they're set by the platform on platform-authored actions, never by an agent:
>
> - **`read`** — classifies an action as a pure read, so the runtime skips its
>   per-invoke change-stream (`_outbox`) row. Set by the auto-CRUD `read`/`list`
>   generators and the registry read actions. Your declared actions are always
>   treated as mutations (the change stream is retained).
> - **`reaction_safe`** — allowlists an action as a valid reaction `via` target
>   (it runs under the internal `system` principal). Only platform delivery
>   actions (e.g. `alerts.send`) carry it; `reactions.declare` rejects an
>   unvetted `via`.
> - **`sensitive_fields`** — names input params (a token, an API key) scrubbed to
>   `[redacted]` in the audit copy. Used by platform actions like
>   `connections.create`; the handler still receives the real value.
> - **`persistArgs`** — a pure projection that slims what's written to the audit /
>   outbox record (used by `catalog.apply` to store a bundle digest, not the
>   ~35 KB bundle). It's a function on the ActionType, not a JSON field.
>
> A custom action gets sensible defaults for all four automatically; you don't —
> and can't — set them in the declaration.

## Example A — a simple transition action (`deal.send_proposal`)

When the job is a state flip plus one timestamp, the handler is a single
`update`. The state column write is auto-injected by the FSM — **never**
put `stage` in your `set`.

```json
{
  "name": "deal.send_proposal",
  "description": "Send a proposal for an open deal.",
  "object_type": "deal",
  "input": { "deal_id": { "type": "string" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Deal.stage",
    "from": ["open"],
    "to": "proposal_sent"
  },
  "handler": {
    "steps": [
      { "op": "update", "object": "deal", "where": { "id": "$input.deal_id" }, "set": { "title": "$input.deal_id" } }
    ],
    "returns": { "deal_id": "$input.deal_id" }
  }
}
```

The same shape works for `deal.win` (`from: [proposal_sent]`,
`to: won`) and `deal.lose` (`from: [open, proposal_sent]`, `to: lost`) —
change the `transition` marker and the column you stamp.

## Example B — a multi-table atomic action (`deal.win_and_activate`)

Read a row, write across two ObjectTypes, return ids — all in one
transaction. Here winning a deal also activates its customer.

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
      { "op": "read",   "object": "deal",     "where": { "id": "$input.deal_id" },    "as": "deal" },
      { "op": "update", "object": "deal",     "where": { "id": "$deal.id" },          "set": { "amount": "$deal.amount" } },
      { "op": "update", "object": "customer", "where": { "id": "$deal.customer_id" }, "set": { "status": "active" } }
    ],
    "returns": {
      "deal_id": "$deal.id",
      "customer_id": "$deal.customer_id"
    }
  }
}
```

The `read` step is load-bearing: it binds `$deal` (so `$deal.customer_id`
is available downstream) and gives you `NOT_FOUND` semantics if the id is
wrong — an `update` alone would silently no-op on a missing row.

## Invoke it — three surfaces

**CLI:**

```sh
canon actions invoke actions.declare --input @deal-win-and-activate.json
```

**REST** (`{ input }` wraps the whole declaration):

```sh
curl -X POST https://api.canonify.app/v1/actions/actions.declare \
  -H "Authorization: Bearer $CANON_API_KEY" \
  -H "Content-Type: application/json" \
  -d @deal-win-and-activate-rest.json   # { "input": { ...the declaration... } }
```

**MCP** — the `actions.invoke` tool:

```json
{
  "name": "actions.invoke",
  "arguments": {
    "name": "actions.declare",
    "args": { "name": "deal.send_proposal", "description": "Send a proposal for an open deal.", "object_type": "deal", "input": { "deal_id": { "type": "string" } }, "governance": { "requires_approval": false, "allowed_principals": ["user", "agent", "service_account"] }, "idempotency": "none", "transition": { "state_machine": "Deal.stage", "from": ["open"], "to": "proposal_sent" }, "handler": { "steps": [ { "op": "update", "object": "deal", "where": { "id": "$input.deal_id" }, "set": { "title": "$input.deal_id" } } ], "returns": { "deal_id": "$input.deal_id" } } }
  }
}
```

## Run and inspect your new action

```sh
canon deal.send_proposal --deal-id dl_…    # per-action sugar
canon actions describe deal.send_proposal --json   # the full declaration back
canon deal.send_proposal --explain          # same payload, shortcut
```

## Idempotency contract

| Submission | Outcome |
|---|---|
| Same `name`, same JSON | Free replay (with key) or no-op success |
| Same `name`, **different** JSON | `INVALID_STATE/declaration_changed` — use `actions.update` |

## Failure modes

| Symptom | Code | Recovery |
|---|---|---|
| Handler references unknown ObjectType / step / property | `VALIDATION/typecheck_failed` | declare/refine it first; read the cited step |
| Handler writes the FSM-bound state column | `VALIDATION/direct_state_write` | remove it from `set` — the runtime auto-writes it |
| Transition marker drifts from the machine | `VALIDATION/<drift_code>` | align `transition.from` / `transition.to` with Chapter 3 |
| Same name, different JSON | `INVALID_STATE/declaration_changed` | use `actions.update` |

## Next

→ [Chapter 5 — Compose a ViewSpec](viewspec-quick-start.md): give your
app its rendered surfaces.
