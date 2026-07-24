# Envelope shape and error handling

Every runtime call — every surface, write or read, success or failure —
returns the same envelope. This file pins the exact shape and shows real
success and error envelopes taken from the live slice-1 fixtures and the
conformance runner's normalization rules.

## The envelope type

The base shape (`packages/canon/src/types.ts`, `Envelope<T>`):

```ts
interface Envelope<T = unknown> {
  data: T;                          // typed object | list | rows (null on error)
  audit_event_id: string;          // non-empty even on reads
  approval_status:                 // governance status for the call
    'not_required' | 'pending' | 'approved' | 'rejected';
  knowledge_used: readonly string[];   // [] until Phase 22 populates it
  outcome_signals: readonly unknown[]; // [] until post-call feedback populates it
  // optional, additive fields (present only when relevant):
  request_id?: string;             // _approval_requests.id when Stage 4 fired
  approved_by?: { kind, id };      // set by actions.approve / actions.reject
  next_cursor?: string | null;     // keyset-pagination cursor for list
  knowledge?: ...;                 // Knowledge roster bundled onto objects.get
}
```

The runtime overlay (`packages/canon/src/runtime.ts`) adds the error field:

```ts
interface Envelope<T> extends BaseEnvelope<T> {
  error?: RuntimeError;            // present only on failure
}

interface RuntimeError {
  code:                            // stable codes a client can switch on
    'VALIDATION' | 'NOT_FOUND' | 'FORBIDDEN' | 'INTERNAL_ERROR' |
    'INVALID_STATE' | 'CONFLICT';
  message: string;                 // human-readable
  issues: unknown[];               // field-path issues; populated for VALIDATION
  details?: Record<string, unknown>;  // e.g. { current_state, allowed_from } for INVALID_STATE
  suggestion?: DenialSuggestion;   // knowledge.contribute hint on FORBIDDEN/INVALID_STATE denials
}
```

`CONFLICT` is the concurrency / lifecycle rejection — a write that changed
nothing because the world moved under it. It surfaces from a `ConflictError`
and always carries a machine-readable `details` bag the client can switch on
instead of parsing `message`:

- **Compound-write freeze** (`parts_frozen_when`): a part edit blocked because
  the root row is in a frozen state. `details` is
  `{ reason: 'FROZEN', object_type, column, frozen_values, current_value }` —
  test `details.reason === 'FROZEN'` to tell a freeze from the stale-version
  case below (which has no `reason`).
- **Stale-version compound write** (optimistic CAS): a `ConflictError` whose
  `details` carries no `reason` — the root version moved since you read it, so
  re-read and retry.
- **`catalog.apply` stale revision**: you pinned `expectedRevision` and it no
  longer matches. `details` is
  `{ code: 'STALE_REVISION', current, expected }` — re-export the canon, rebuild
  your desired state, re-apply.
- **`catalog.apply` already running**: another apply holds the per-org lease.
  `details` is `{ code: 'APPLY_IN_PROGRESS' }` — kept distinct from
  `STALE_REVISION` so a client can tell "retry shortly" from "re-read first".

## A real success envelope

The slice-1 `customer.create` fixture
(`packages/canon/src/actions/customer/create.ts`) invokes the action as the
fixture principal `{ kind: 'user', id: 'usr_fixture00000000000000000', org_id:
'org_test' }` with args `{ name: 'Alice', email: 'alice@example.com', org_id:
'org_test' }`. The runtime mints an id and timestamps, then returns:

```jsonc
{
  "data": {
    "id": "cust_01HW…",                 // minted prefixed-ULID
    "name": "Alice",
    "email": "alice@example.com",
    "org_id": "org_test",
    "created_at": "2026-05-11T12:00:00.000Z",
    "updated_at": "2026-05-11T12:00:00.000Z"
  },
  "audit_event_id": "audit_01HW…",      // minted per call
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

This exact envelope comes back over REST (`POST /v1/actions/customer.create`
body `{ "input": { "name": "Alice", "email": "alice@example.com", "org_id":
"org_test" } }`), CLI (`canon customer.create --name Alice --email
alice@example.com --org_id org_test --output json`), and MCP (tool
`actions.invoke` with `{ name: "customer.create", args, principal }`) — modulo
the non-deterministic fields below.

## What the conformance runner normalizes (and what it doesn't)

Two surface calls of the same fixture cannot mint the same id, audit id, or
timestamp, so the runner masks exactly those before comparing. From
`tests/conformance/runner.ts`:

- top-level `audit_event_id` → `"<audit_event_id>"`
- any prefixed-ULID string → `"<id:cust>"`, `"<id:audit>"`, etc. The masked
  prefixes include `cust_`, `audit_`, `aud_`, `evt_`, `out_`, `idem_`, `mig_`,
  `re_`, `prep_`, `know_`, `kdrift_`, `notif_`, `grp_`, `gm_`.
- any value under a timestamp key (`created_at`, `updated_at`, `started_at`,
  `completed_at`, `occurred_at`, `timestamp`) → `"<timestamp>"`
- any ISO-8601-looking string anywhere → `"<timestamp>"`

**Everything else is compared byte-for-byte.** So the normalized
`customer.create` envelope the runner asserts is identical across all three
surfaces is:

```jsonc
{
  "data": {
    "id": "<id:cust>",
    "name": "Alice",
    "email": "alice@example.com",
    "org_id": "org_test",
    "created_at": "<timestamp>",
    "updated_at": "<timestamp>"
  },
  "audit_event_id": "<audit_event_id>",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

Note what is *not* masked: `name`, `email`, `org_id`, `approval_status`, and
the two empty arrays. A surface that dropped `org_id`, renamed a field, or
returned `approval_status: 'pending'` would fail — which is exactly the kind of
drift the negative half of the gate injects (it swaps `name` for `"DRIFT"`,
which survives normalization, and asserts the runner flags it).

## A real error envelope

Many slice-1 fixtures deliberately target missing rows so all three surfaces
produce the same domain error. For example, the `customer.update_email`,
`knowledge.archive`, `knowledge.confirm`, and `files.detach` fixtures reference
ids that don't exist in the fresh conformance runtime, so every surface returns
the identical `NOT_FOUND` envelope:

```jsonc
{
  "data": null,
  "audit_event_id": "<audit_event_id>",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": [],
  "error": {
    "code": "NOT_FOUND",
    "message": "…",                  // deterministic, compared byte-for-byte
    "issues": []
  }
}
```

The critical isomorphism detail: **REST returns this with a 4xx status, but the
body is still a full envelope.** The conformance runner reads the body
regardless of status, because it compares *envelopes*, not HTTP status codes. A
client should do the same — switch on `error.code`, not on the transport's
status convention. A `VALIDATION` error additionally populates `issues` with
field-path detail; an `INVALID_STATE` error populates `details` with
`{ current_state, allowed_from }`.

## Response header: `X-Catalog-Invalidate`

A `POST /v1/actions/<name>` whose invoke **mutated the catalog** (a
`catalog.*` / `schema.*` verb, `view_specs.declare`,
`state_machines.register`, a refine that overlays an ObjectType, …) carries
the response header `X-Catalog-Invalidate: catalog`. It signals a caching
client — the `canon` CLI caches the catalog for ~15 minutes — to drop its
cache so the next `actions describe` / `actions list` / `objects list-types`
re-fetches the fresh schema. Read-only invokes omit the header. Direct REST
callers that don't cache can ignore it; the envelope body is unchanged. It
closes the "invoke accepted my new field but `describe` still shows the old
input schema" window.

## Discoverability is isomorphic too

The same envelope discipline extends to discovery. `GET /v1/actions`,
`canon actions list`, and the MCP `actions.list` tool all return the *live*
catalog, not a static file — so an action is discoverable identically on all
three surfaces, and cross-surface disagreement about what exists is detectable
at runtime.
