# Handler DSL cookbook

Copy-paste-ready handler examples. Every JSON block in this file decodes
cleanly against the `HandlerDsl` effect/Schema in
[`packages/canon/src/handler-dsl/schema.ts`](../../../../packages/canon/src/handler-dsl/schema.ts).
If it decodes here it executes inside the runtime; the test
`tests/integration/canonify-skill-content.test.ts` enforces this by
decoding the first JSON block on this page against `HandlerDsl`.

The handler DSL is defined in
[Phase 17.5 §1](../../../phases/17.5-declarative-handlers.md#1-the-handler-dsl--vocabulary)
(vocabulary) and
[§2](../../../phases/17.5-declarative-handlers.md#2-the-dsl-is-itself-a-zod-schema--packagescanonsrchandler-dslschemats)
(the effect/Schema). The executor contract — what runs inside the tx — is
[§4](../../../phases/17.5-declarative-handlers.md#4-the-runtime-executor--packagescanonsrchandler-dslexecutorts).

---

## Reference grammar — one screen

| Ref | Meaning |
|---|---|
| `$input.<field>` | A field on the action's schema-decoded input |
| `$<step>.<field>` | A field bound by a prior `read` or `insert` step |
| `$now` | Runtime clock — ISO 8601 UTC |
| `$principal.id` / `.org_id` / `.kind` | Caller identity |
| `$actor.<attr>` | A server-set external-principal attribute — resolves ONLY from `ExternalPrincipal.attributes`, never the payload; fails closed (`VALIDATION`) for a non-external principal or an absent/empty attribute. Required to address an `actor_attr`-target action's row |
| `"literal"`, `42`, `true`, `null` | Plain JSON literals |

Refs resolve on **both sides of a predicate**: the comparison value of a
`when`/`assert` operator (`eq`/`neq`/`gt`/`gte`/`lt`/`lte`, and each `in`
member) may itself be a ref — the canonical cut-off guard is
`{ "ref": "$input.day_id", "lt": "$now" }`, which compares against the
resolved clock instant, not the literal characters `"$now"`.

A string starting with `$` is **always** a reference; literals that begin
with `$` are written `$$literal`. Refs must match
`/^\$(input|principal|now|[a-z_][a-z0-9_]*)(\.[a-z_][a-z0-9_]*)*$/` —
lowercase-snake heads and segments only.

Step kinds:

| op | Effect | Binds via `as`? |
|---|---|---|
| `read` | SELECT one row; throws `NOT_FOUND` if missing | required |
| `insert` | INSERT a row; executor stamps `id`, `created_at`, `updated_at` | optional |
| `update` | UPDATE by `where`; executor stamps `updated_at` | — |
| `delete` | DELETE by `where`; respects `on_delete` cascade on the ObjectType's links. A `where` matching zero rows is a benign no-op (not `NOT_FOUND`), so the idempotent "delete pending X, then re-insert" pattern succeeds on first run | — |

Notably **absent** from v1: `if`/`else`, loops, expressions inside
values (no `$now + 1d`, no concat), raw SQL. The constraint is
deliberate; see Phase 17.5 §1.

---

## Example 1 — `lead.convert` (the centerpiece)

The canonical multi-table atomic write. Lifted verbatim from
[Phase 17.5 §10 worked example](../../../phases/17.5-declarative-handlers.md#worked-example-leadconvert-declared-by-an-agent)
and [`tests/scenarios/minimal-crm/session.yaml`](../../../../tests/scenarios/minimal-crm/session.yaml).

`POST /v1/actions/actions.declare` body:

```json
{
  "name": "lead.convert",
  "description": "Convert a qualified lead into a Customer + primary Contact.",
  "object_type": "lead",
  "input": {
    "lead_id": { "type": "string" }
  },
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
      {
        "op": "read",
        "object": "lead",
        "where": { "id": "$input.lead_id" },
        "as": "lead"
      },
      {
        "op": "insert",
        "object": "customer",
        "values": {
          "name": "$lead.name",
          "email": "$lead.email",
          "org_id": "$principal.org_id"
        },
        "as": "customer"
      },
      {
        "op": "insert",
        "object": "contact",
        "values": {
          "customer_id": "$customer.id",
          "name": "$lead.name",
          "email": "$lead.email",
          "role": "primary"
        },
        "as": "contact"
      },
      {
        "op": "update",
        "object": "lead",
        "where": { "id": "$input.lead_id" },
        "set": {
          "customer_id": "$customer.id",
          "converted_at": "$now"
        }
      }
    ],
    "returns": {
      "lead_id": "$input.lead_id",
      "customer_id": "$customer.id",
      "contact_id": "$contact.id"
    }
  }
}
```

What happens at `runtime.invoke('lead.convert', { lead_id: 'ld_…' })`:

1. Idempotency check, governance check, schema-decode input, open tx
2. Phase 18.5 transition guard asserts `lead.status === 'qualified'`
3. Executor runs the four steps in order inside the same tx:
   - `read lead` → binds `$lead`
   - `insert customer` (executor stamps `id` = `cus_…` from the
     ObjectType prefix, `created_at` = `updated_at` = `$now`)
   - `insert contact` referencing `$customer.id` from step 2
   - `update lead` with `customer_id` and `converted_at`
4. Phase 18.5 post-update auto-writes `lead.status='converted'` —
   **the handler must not write it** (`actions.declare` rejects with
   `VALIDATION/direct_state_write` if you try)
5. Phase 18.5 on_enter[`converted`] hook enqueues the
   `email.send_template` welcome row into `_outbox`
6. Audit row written with `status='success'`, outbox rows committed,
   tx commits
7. Envelope returns `{ data: { lead_id, customer_id, contact_id },
   audit_event_id }`

Failure modes for this handler map cleanly to ActionDomainError kinds —
see [`error-codes.md`](error-codes.md).

---

## Example 2 — Simple state-transition update (`lead.contact`)

When the action's job is a state flip plus a single timestamp stamp,
the handler is one step. The state column update is auto-injected by
Phase 18.5 — never write it in your handler.

```json
{
  "name": "lead.contact",
  "description": "Mark a new lead as contacted.",
  "object_type": "lead",
  "input": {
    "lead_id": { "type": "string" }
  },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": {
    "state_machine": "Lead.status",
    "from": ["new"],
    "to": "contacted"
  },
  "handler": {
    "steps": [
      {
        "op": "update",
        "object": "lead",
        "where": { "id": "$input.lead_id" },
        "set": { "contacted_at": "$now" }
      }
    ],
    "returns": {
      "lead_id": "$input.lead_id"
    }
  }
}
```

The same shape works for `lead.qualify`, `lead.disqualify`,
`deal.send_proposal`, `deal.win`, `deal.lose` — change the `transition`
marker and the timestamp column. The seven CRM transition Actions in
`tests/scenarios/minimal-crm/session.yaml` are all this pattern.

---

## Example 3 — Insert-only with `returns`

Sometimes the action's job is to materialise a single row and hand the
caller its id. No `read` step needed.

```json
{
  "name": "note.create_for_customer",
  "description": "Attach a free-form note to a customer.",
  "object_type": "note",
  "input": {
    "customer_id": { "type": "string" },
    "body": { "type": "string" }
  },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "per-key",
  "handler": {
    "steps": [
      {
        "op": "insert",
        "object": "note",
        "values": {
          "customer_id": "$input.customer_id",
          "body": "$input.body"
        },
        "as": "note"
      }
    ],
    "returns": {
      "note_id": "$note.id",
      "created_at": "$note.created_at"
    }
  }
}
```

`$note.id` is the executor-generated id (prefix derived from the
ObjectType — `nt_…` for `note`). `$note.created_at` is the `$now`
value the executor stamped on insert.

If the caller doesn't need the row id back, drop the `as` and the
`returns` and the action returns `undefined` in `envelope.data`.

---

## Example 4 — `read` then `update` (compose-by-id)

A common shape for "validate that the row exists, then mutate" without
needing a full multi-table effect. The `read` is load-bearing — without
it the agent has no way to assert `NOT_FOUND` semantics before the
`update` (which would silently no-op on a missing id).

```json
{
  "name": "deal.assign_owner",
  "description": "Assign an owner to an existing deal.",
  "object_type": "deal",
  "input": {
    "deal_id": { "type": "string" },
    "owner_id": { "type": "string" }
  },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent"]
  },
  "idempotency": "none",
  "handler": {
    "steps": [
      {
        "op": "read",
        "object": "deal",
        "where": { "id": "$input.deal_id" },
        "as": "deal"
      },
      {
        "op": "update",
        "object": "deal",
        "where": { "id": "$deal.id" },
        "set": { "owner_id": "$input.owner_id" }
      }
    ],
    "returns": {
      "deal_id": "$deal.id",
      "owner_id": "$input.owner_id"
    }
  }
}
```

---

## Anti-patterns the type-checker will reject

These declarations get rejected at `actions.declare` time with a
`VALIDATION` envelope citing the offending step. See Phase 17.5 §3.

1. **Unknown ObjectType.**
   `{ "op": "read", "object": "leed", ... }` → `unknown object: leed`.
2. **Unknown bound step.**
   `{ "values": { "customer_id": "$customer.id" } }` when no prior
   step binds `as: "customer"` → `unknown ref: $customer.id`.
3. **Unknown field on a bound step.**
   `"name": "$lead.foobar"` when `lead` has no `foobar` property →
   `lead has no property "foobar"`.
4. **Writing the FSM-bound state column.**
   When the declaration carries a `transition` marker, an `update`
   step's `set` map MUST NOT include the state column — the runtime
   auto-writes it. Surfaces as `VALIDATION/direct_state_write`.
5. **Empty `steps` array.**
   `"handler": { "steps": [] }` → the schema rejects at the surface with
   `handler must declare at least one step`.
6. **Malformed ref.**
   `"$Input.lead_id"` (capital `I`) → the schema rejects as not matching the
   ref regex.

---

## Discovering what's registered

Before declaring a new ActionType, list what's there:

```sh
canon actions list --json
canon actions describe lead.convert --json
canon objects list-types
canon schema describe --object lead
```

`actions describe <name>` returns the full declaration — input JSON
Schema, governance, transition marker, fixture. `--explain` on any
per-action sugar command is the shortcut to the same payload.
