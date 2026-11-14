# E-signature actions

Platform actions every org has — you **invoke** them, never declare them. Three
object types: `signature_request` (one signer), `signature_envelope` (ordered
signers) and the immutable `signature`, plus the per-member saved
`member_signature`. Every per-record action declares its `subject`
(`request_id`, `envelope_id`, `signature_id`), so a row or record binds it;
the creates have none. Both `status` columns are platform state machines
(`signature_request.lifecycle`, `signature_envelope.lifecycle`: state labels,
colors, legal moves), and the per-record actions carry
`presentation.showWhen` preconditions, so a record offers only what can
succeed there. The browser side — the external `/_sign` page, the
magic-link + PIN session, the `/_canon/sign-token` and `/_canon/download-token`
proxies — is platform behaviour you get without authoring anything. Examples
below are invoke payloads.

## Who calls what

| Caller | Authenticates by | Actions |
|---|---|---|
| **external** signer (a non-member reached by email) | redeeming the emailed `link_token` | `signature_request.view` / `.sign` / `.decline`; for the matching signer `signature.certificate`, `signature_envelope.download` |
| **member** (`user` / `agent`) | the session | `signature_request.create` / `.send`, `signature.verify` / `.certificate`, `signature_envelope.create` / `.send` / `.void` / `.countersign` / `.decline` / `.my_pending_countersigns` / `.download`, `member_signature.set` / `.get` |

These are the only platform actions that admit the `external` kind, and an
app's access manifest cannot widen that. `.view` / `.sign` / `.decline` admit
**only** `external` (discovery's `allowed_principals` lists member kinds only,
so theirs is empty and no member menu offers them); `.countersign`, the
member `.decline`, `.void`, `.my_pending_countersigns` and `member_signature`
admit `user` / `agent`; the rest also admit `service_account`. Each handler guards the same kinds: a member calling an
external-only action, or the reverse, is `FORBIDDEN`. Test the signer path with
an `external` principal and the guard with a member.

**`link_token` is a bearer credential.** Possession of a token whose SHA-256
matches the stored hash is the identity proof (the raw token is never stored).
Every action that takes one redacts it to `"[redacted]"` in the audit row and
the outbox payload (failed attempts too); a `signature_image` is likewise kept
out of the trail.

**Signing evidence is server-derived.** Client IP, User-Agent, auth method and
signing-session id come only from the request context the hosted-apps proxy
fills; they are never input fields. A bare invoke records them as `null`.

## `signature_request` — one signer

```
pending → draft → sent → viewed → signed
                               ↘ declined
                  (any open)   ↘ voided
```

`pending` is an envelope slot not yet its turn (a standalone request starts
`draft`). `signed`, `declined`, `voided` are terminal and freeze the request's
parts (`parts_frozen_when`), so no signature can be written to it.

**`.create`** (member) — persists a `draft`, does not send.

```jsonc
{
  "signer_email": "signer@example.com",   // required
  "signer_name": "Sam Signer",            // optional
  "document_content_hash": "<64-hex>",    // required (a governance gate enforces it: VALIDATION if missing)
  "document_file_id": "file_…",           // optional: the unsealed PDF sealed at sign time
  "expires_at": "2026-07-15T00:00:00Z",   // optional; enforced at view/sign
  "require_step_up": false                 // optional: require the emailed PIN before signing
}
```

**`.send`** (member) — `draft|sent → sent`; mints the signing token and emails
one invitation. Input `{ request_id, document_title?, sender_name? }`; its link
is always your org's portal `https://apps.canonify.app/<org>/_sign` (a
`signing_url_base` key is refused, `signing_url_base_refused`). It is the
machine's transition, so any other status → `INVALID_STATE`.

**`.view`** (external, read) — `{ request_id, link_token }`. Returns the
unsealed PDF as a data-URL plus the request fields (`require_step_up` tells the
portal whether to show the PIN step); `sent → viewed` on open; a `signed`
request also returns the sealed receipt. Viewable from `sent | viewed | signed`;
bad token, wrong actor, expired link, unsent or declined request → typed errors.

**`.sign`** (external) — writes the immutable `signature` (its `record_hash`
binds the evidence), PAdES-seals the document plus a Certificate of Completion
with the org's certificate, and moves the request to `signed`.

```jsonc
{
  "request_id": "sigreq_…",
  "link_token": "<emailed token>",
  "signer_name": "Sam Signer",
  "signature_image": "data:image/png;base64,…",   // data-URL or bare base64
  "consent": { "acknowledged": true, "consent_version": "v1" }
}
```

Consent is mandatory: `acknowledged` must be `true` and `consent_version` a
known statement (`v1`); the server resolves the statement text itself and binds
text and version into `record_hash` (unknown version → `VALIDATION`). Signable
from `sent | viewed`. With `require_step_up`, a link-only token is
`FORBIDDEN/STEP_UP_REQUIRED` — only the PIN session may sign. One signature per
request; a re-sign is `CONFLICT`.

**`.decline`** (external) — `{ request_id, link_token, decline_reason? }`. A
standalone request goes `declined`. An envelope slot goes `declined`, **its
envelope goes `declined` and every other open slot `voided`**. Idempotent
(`{ already_declined: true }`). The reason is kept in the audit args, not a
column.

## `signature` — verifying

**`signature.verify`** (member) — `{ signature_id }` → `{ valid, reasons }`,
`valid` when `reasons` is empty. It re-derives the record hash and the document
hash and checks the PAdES seal; reasons are stable strings (`"record_hash
mismatch"`, `"document content hash mismatch"`, `"seal invalid: …"`, including
`"seal invalid: sealed document not found"` for a signature from before
sealing). An unknown id and a signature the member may not read return the same
`NOT_FOUND`, so verify is no existence oracle.

**`signature.certificate`** (member who can read the request, or the matching
external signer) — `{ signature_id }` → `{ signature_id, certificate_data_url,
source }`, `source` `"sealed"` or `"rendered"` (fallback for an unsealed legacy
row). A member who cannot read the request → `FORBIDDEN`; unknown id →
`NOT_FOUND`.

## `signature_envelope` — ordered signers

```
draft → in_progress → completed
                   ↘ declined
                   ↘ voided
```

`create` lands `in_progress` (`draft` is unused). Each slot is a
`signature_request` (`envelope_id`, `order_index`). Terminal envelopes never
advance.

**`.create`** (member):

```jsonc
{
  "document_content_hash": "<64-hex>",   // required (gate)
  "document_file_id": "file_…",          // required (gate)
  "title": "Mutual NDA",
  "signers": [                            // array order = signing order; ≥ 1
    { "party_label": "Buyer",  "channel": "external", "email": "buyer@example.com" },
    { "party_label": "Seller", "channel": "member",   "member_account_id": "acct_…" }
  ]
}
```

Each signer needs a `party_label`; `external` needs `email`, `member` needs
`member_account_id`. Slot 0 starts `draft`, later slots `pending`. No send.

**`.send`** (member) — `{ envelope_id, sender_name?, document_title? }`
(`sender_name` is reused as the envelope advances; every link is on the
platform portal). Opens the current turn: an external slot gets its invitation emailed; a member slot is
left active with no email (the member countersigns from the dashboard). Not
`draft|in_progress` → `INVALID_STATE` (the machine's transition guard).

**`.countersign`** (member) — `{ request_id, signer_name, signature_image,
consent }`, same consent contract as `.sign`. Writes the same evidence with
`auth_method: 'member_session'` and `signer_actor_id: member:<account_id>`. The
slot must be a member slot assigned to the caller, active (`draft`) and its turn
(`CONFLICT/NOT_YOUR_TURN` otherwise). Advances the envelope. Offered only on an
open member slot (`showWhen`: `channel` `member`, `status` `draft`), as is the
member `.decline`.

**Completion is automatic** — there is no `.complete` action. When the last
slot signs, the sign/countersign handler assembles the document plus a
multi-signer certificate, seals it once as `completed_file_id`, marks the
envelope `completed` and emails every party a download link.

**`.download`** (party only) — `{ envelope_id }` → `{ envelope_id, status,
completed_file_id, document_data_url }`. Only a signer of a slot (external by
actor id, member by account id); anyone else `FORBIDDEN`; unknown `NOT_FOUND`;
not completed `CONFLICT/NOT_COMPLETED` (offered only on a `completed`
envelope).

**`.my_pending_countersigns`** (member) — `{}` → `{ pending: [{ request_id,
envelope_id, party_label, order_index, title, document_content_hash,
document_file_id }] }`: the caller's own active member slots, bound server-side
to the principal.

**`.decline`** (member) — `{ request_id, decline_reason? }`; same guards as
countersign, same propagation as the external decline. Idempotent.

**`.void`** (member, sender side) — `{ envelope_id, void_reason? }` on a
`draft|in_progress` envelope: it goes `voided` and every open slot is voided.
Idempotent (`{ already_voided: true }`); `completed` / `declined` → `CONFLICT`
(offered only on `draft|in_progress`). Only the envelope's creator or an org
admin may void (`FORBIDDEN` otherwise; an envelope with no recorded creator
stays any-member).

## `member_signature` — the saved signature

A convenience store reused on every countersign, not evidence (no consent, no
`record_hash`).

- **`.set`** — `{ signer_name, signature_image, source: "typed" | "drawn" }` →
  `{ account_id, signer_name, source, signature_image_content_hash, created_at,
  updated_at }`. Upserts one row per member.
- **`.get`** — `{}` → `{ has_signature, signer_name, source, signature_image,
  signature_image_content_hash, created_at, updated_at }` (all `null` with
  `has_signature: false` when unset). Self-scoped.
