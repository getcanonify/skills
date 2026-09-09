# Outbox actions — the governed effect-queue operator surface

`outbox.inspect` and `outbox.requeue` are **platform primitives** —
registered unconditionally in the catalog (`packages/canon/src/catalog.ts`),
not declared with `actions.declare`. You **invoke** them like any other
action (CLI / REST / MCP); you never author them. They give an operator a
governed view onto the org's deferred-effect queue (`_outbox`) — the same
queue that delivers emails, webhooks, FSM `on_enter`/`on_exit` hooks, and
Reaction/schedule fan-out — and a governed way to retry what got stuck,
instead of a hand-written SQL statement against the database.

Both require **`manage` authority** (owner/admin; the `member` role baseline
is read + run and does not include queue operations). Every shape below is
grounded in `packages/canon/src/actions/outbox/outbox.ts`.

---

## `outbox.inspect` — read-only queue health + rows

Marked `read: true` — looking at the queue never writes a change-stream row
of its own.

```jsonc
{}                                    // defaults: every non-terminal row first, limit 20
{ "status": "failed" }                // narrow to one queue state
{ "status": "claimed", "limit": 50 }  // limit caps at 100
```

`status` is one of `"new"` | `"claimed"` | `"failed"`. Omit it to see every
**non-terminal** row (`claimed` before `new`, oldest first) followed by
deadletters — the order an operator triages in.

Response shape:

```jsonc
{
  "health": {
    "pending": 0,           // status='new' rows, due or waiting on a backoff
    "due_pending": 0,       // the subset of pending whose next_attempt_at has arrived
    "stale_claimed": 0,     // claims whose lease EXPIRED — the owner died mid-delivery
    "live_claimed": 0,      // claims still inside their lease — a worker is delivering them
    "failed": 0,            // deadlettered rows — terminal until an operator requeues
    "oldest_pending_at": null,      // created_at of the oldest non-terminal row, or null
    "oldest_pending_age_ms": null   // its age at read time, in ms; null when the queue is empty
  },
  "rows": [
    {
      "id": "out_…",
      "action_id": "email.send_template",
      "status": "failed",
      "kind": "effect",
      "attempts": 3,
      "next_attempt_at": null,
      "claimed_by": null,
      "claimed_at": null,
      "lease_expired": false,
      "created_at": "2026-08-10T12:00:00.000Z",
      "completed_at": null,
      "idempotency_key": "sched:reminder:row_1:2026-08-10T12",
      "last_error": "Provider rejected recipient address"
    }
  ],
  "truncated": false   // true when more rows matched than `limit` returned
}
```

**The row `payload` never comes back.** A row's payload is a prior action's
INPUT, and that action may declare `sensitive_fields` — returning the raw
payload would build a second, ungoverned read path around that redaction.
What an operator needs to act is the row's identity, its retry state, and
its lease state, plus (for a deadletter) `last_error` — so that is exactly
what this surface returns.

**`last_error` is the one narrow way information about the input comes back
in.** `markFailed` records a runtime failure message verbatim, and a
validation-class message can quote the input that failed. Two bounds apply
before it leaves this surface:

1. The target action's own **declared `sensitive_fields`** are redacted from
   the message (the same contract the audit trail's redaction honours) —
   longest value first, so a value that contains another isn't half-masked.
2. The message is **capped at 240 characters**, whether or not any of it was
   declared sensitive.

The honest residual: a message that happens to quote an *undeclared* input
field can still show a fragment of it, because nothing marks that field as
secret anywhere in the catalog. If that matters for an action you own, the
fix is to declare the field `sensitive_fields` — which also fixes the audit
trail's redaction for the same reason.

---

## `outbox.requeue` — put a stuck effect back on the queue

```jsonc
{ "id": "out_…" }                              // restart the retry ladder (default)
{ "id": "out_…", "reset_attempts": false }      // keep the existing attempt count
```

`reset_attempts` defaults to **`true`**: an operator requeueing a deadletter
almost always wants a fresh run at it, and a row that kept
`attempts = MAX_ATTEMPTS` would deadletter again on its very next failure.
Pass `false` to preserve the count instead — e.g. when requeueing a row you
expect to fail the same way, so it walks the retry ladder to a deadletter
rather than looping silently.

Response shape:

```jsonc
{
  "id": "out_…",
  "status_before": "failed",   // the status the handler read before acting
  "requeued": true,
  "attempts": 0,                // attempts AFTER the requeue (0 when the ladder restarted)
  "next_attempt_at": "2026-08-17T10:00:00.000Z"
}
```

When nothing was requeued, `requeued: false` and `reason` names why:

- **`already_queued`** — the row is already `status='new'`. A clean no-op,
  not an error, so a retried operator call (or two operators at once) is
  harmless.
- **`raced`** — the row moved between the read and the conditional update (a
  drain claimed or reclaimed it in between). The update is a
  compare-and-swap against the exact state the handler read; anything but
  exactly one matched row reports `raced` rather than assuming success.

Three things `outbox.requeue` refuses, each with a typed `VALIDATION` /
`NOT_FOUND` failure naming why:

- **A row inside a LIVE claim** (`claim_live`) — a worker is delivering it
  right now; taking it back would let two workers deliver the same effect.
  The message names when the lease expires and becomes recoverable (the
  drain reclaims an expired lease automatically, with no operator action).
- **A row already `done`** (`already_delivered`) — re-running a completed
  effect is a duplicate send, and this action exists to recover
  *undelivered* work, not to replay delivered work.
- **A `kind='event'` row** (`not_an_effect`) — an event-log row is never
  claimed by a worker, so "requeueing" one would only move a log entry, not
  retry a delivery.

`markFailed` wraps a failed row's payload as `{ payload, _failure }` to carry
the recorded error; `outbox.requeue` unwraps that back to the original
payload before putting the row back on the queue, so the redelivery decodes
as the target action's real input instead of the failure wrapper.

## See also

- [actions-input-governance.md](actions-input-governance.md) — idempotency
  postures generally, and how the outbox drain's `mergeIdempotencyKey` stamps
  a stable `idempotency_key` onto a re-invoked effect (the same key you see
  on an `outbox.inspect` row).
- Base skill: `canon skill --name canonify instructions`.
