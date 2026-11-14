# Outbox actions — inspect and requeue the effect queue

`outbox.inspect` and `outbox.requeue` are platform actions over `_outbox`, the
queue that delivers emails, webhooks, state-machine hooks and schedule/reaction
fan-out. Both need `manage` (owner / admin).

## `outbox.inspect` (read)

```jsonc
{}                                    // every non-terminal row first, then deadletters; limit 20
{ "status": "failed" }                // "new" | "claimed" | "failed"
{ "status": "claimed", "limit": 50 }  // limit caps at 100
```

Response:

```jsonc
{
  "health": {
    "pending": 0,                 // status='new' (due or backing off)
    "due_pending": 0,             // pending whose next_attempt_at has arrived
    "stale_claimed": 0,           // claim lease expired — the worker died
    "live_claimed": 0,            // a worker is delivering it now
    "failed": 0,                  // deadlettered; terminal until requeued
    "oldest_pending_at": null,
    "oldest_pending_age_ms": null
  },
  "rows": [{ "id": "out_…", "action_id": "email.send_template", "status": "failed",
             "kind": "effect", "attempts": 3, "next_attempt_at": null,
             "claimed_by": null, "claimed_at": null, "lease_expired": false,
             "created_at": "…", "completed_at": null,
             "idempotency_key": "sched:reminder:row_1:2026-08-10T12",
             "last_error": "Provider rejected recipient address" }],
  "truncated": false
}
```

A row's `payload` (a prior action's input) is **never** returned. `last_error`
is the one channel input can leak through, so the target action's declared
`sensitive_fields` are redacted from it and it is capped at 240 characters. A
message quoting an *undeclared* field can still show a fragment — declare the
field in `sensitive_fields` (which also redacts it from the audit trail).

## `outbox.requeue`

```jsonc
{ "id": "out_…" }                          // reset_attempts defaults to true
{ "id": "out_…", "reset_attempts": false } // keep the count: walk to a deadletter again
```

Returns `{ id, status_before, requeued, attempts, next_attempt_at }`. When
nothing was requeued, `requeued: false` with `reason`:

- `already_queued` — already `new`; a harmless no-op.
- `raced` — the row moved between read and the compare-and-swap update.

Refused (`VALIDATION` / `NOT_FOUND`):

- `claim_live` — a worker holds a live lease; the message says when it expires
  (an expired lease is reclaimed automatically).
- `already_delivered` — the row is `done`; requeue recovers undelivered work,
  it never replays delivered work.
- `not_an_effect` — a `kind='event'` row is a log entry, never delivered.

A requeued deadletter is redelivered with its original payload (the recorded
failure wrapper is removed) and keeps its stable `idempotency_key`.
