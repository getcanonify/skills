# E-signature actions — the platform signing surface

The e-signature actions are **platform primitives** — registered
unconditionally in the catalog (`packages/canon/src/catalog.ts`), not declared
with `actions.declare`. You **invoke** them like any other action (CLI / REST /
MCP); you never author them. They cover three object types —
`signature_request` (a single signer), `signature_envelope` (an ordered
multi-signer set), and `signature` (the immutable signed record) — plus a
per-member reusable-signature store (`member_signature`).

This page documents the **action surface an agent invokes**. The browser-portal
side — the external signer's `/_sign` page, the magic-link + PIN session, the
`/_canon/sign-token` and `/_canon/download-token` proxy endpoints — is the
signing-protocol operations doc, not this skill:
[`repo-wiki/operations/signing-protocol.md`](../../../../repo-wiki/operations/signing-protocol.md)
(authored alongside this page). When you need to know *how a signer's browser
redeems a link*, read that; when you need to know *what action to call and with
what input*, read this.

Every shape below is grounded in the action's effect/Schema input in
`packages/canon/src/actions/{signature-request,signature,signature-envelope,member-signature}/`.
The examples are `jsonc` (illustrative invoke payloads), not `actions.declare`
inputs — these actions already exist; you call them.

---

## Two principal populations

The signing actions split cleanly by **who the caller is** — this is the single
most important thing to get right, because the same envelope is touched by both:

- **`external`** (a non-member signer reached by an emailed magic link). The
  external kind is *exempt* from the per-action `allowed_principals` gate (the
  governance schema can't enlist a non-member kind), so each external-only
  handler carries its **own** `principal.kind === 'external'` guard as the real
  enforcer. External actions authenticate by **redeeming a `link_token`** —
  there is no member session. These are: `signature_request.view` / `.sign` /
  `.decline`, and (for the matching signer) `signature.certificate` /
  `signature_envelope.download`.
- **member** (`user` | `agent`, an authenticated org principal). The member's
  session **is** the authorization — no link, no token. These are:
  `signature_request.create` / `.send`, `signature.verify`, the whole
  `signature_envelope` sender side (`.create` / `.send` / `.void`), the member
  countersign side (`signature_envelope.countersign` / `.decline` /
  `.my_pending_countersigns`), and `member_signature.set` / `.get`.

A member reaching an external-only action (or vice-versa) is rejected
`FORBIDDEN` by the handler's kind-guard.

### `link_token` is a bearer credential — and it is redacted from the audit trail

The `link_token` that `signature_request.view` / `.sign` / `.decline` accept is
the **bearer credential** emailed to the signer: possession of a token that
hashes (SHA-256) to the request's stored `signing_token_hash` *is* the identity
proof (the raw token is never stored, only its hash). Because it is a live
credential, every action that takes one declares a **`persistArgs`** projection
that replaces it with `"[redacted]"` before the args are written to the
`_audit_event` row and the `_outbox` change-stream payload (and the same
projection is applied on a *failed*-attempt audit row). `signature_request.sign`
additionally drops the large `signature_image` blob. So a leaked audit row can
never replay a still-valid token. You pass the real token on the wire; it never
lands cleartext at rest.

### Server-derived signing evidence (never client input)

The WHO/WHERE/HOW evidence bound onto an immutable signature — client IP,
User-Agent, auth method, signing-session id — is read **exclusively** from the
runtime-threaded `RequestContext`
(`{ ip?, user_agent?, auth_method?, session_id? }`,
`packages/canon/src/types.ts`), which the hosted-apps proxy populates off the
real HTTP request and the verified external session. It is **never** taken from
the action input — a client cannot smuggle a forged IP / auth_method through
`args`. Absent context (e.g. a bare test invoke) hashes deterministically to
`null`. Do not look for these as input fields; they do not exist.

---

## `signature_request` — the single-signer lifecycle

Status enum (`packages/canon/src/object-types/signature-request.ts`):

```
pending → draft → sent → viewed → signed
                               ↘ declined
                  (any open)   ↘ voided
```

- **`pending`** — an envelope SLOT not yet its turn (a standalone request never
  starts `pending`; it starts `draft`).
- **`draft`** — created, not yet sent.
- **`sent`** → **`viewed`** → **`signed`** is the happy path.
- **`signed` | `declined` | `voided`** are TERMINAL. The object type declares
  `parts_frozen_when: { column: 'status', in: ['signed', 'declined', 'voided'] }`,
  so once terminal the request's parts (its `signature` record) are frozen — no
  signature can be written to a terminal request.

### `signature_request.create` (member)

Persists a `draft` request. Does **not** send.

```jsonc
{
  "signer_email": "signer@example.com",   // required, format email
  "signer_name": "Sam Signer",            // optional display hint
  "document_content_hash": "<64-hex>",    // REQUIRED by a governance gate (see below)
  "document_file_id": "file_…",           // optional; the UNSEALED PDF to seal at sign time
  "expires_at": "2026-07-15T00:00:00Z",   // optional ISO-8601; enforced at view/sign
  "require_step_up": false                 // optional, default false (see step-up)
}
```

`document_content_hash` is schema-*optional* on purpose: a required-field
**governance gate** (the CVR-analogue in `create.ts`) is the enforcer, so a
missing/blank value surfaces a typed `VALIDATION` ("required field
…document_content_hash… is missing") rather than a generic decode error. Lands
`status='draft'`; the signer's actor is a deterministic `pending:<email>`
placeholder bound at sign time.

`require_step_up` (default `false`) opts THIS request into requiring the email
one-time code (the magic-link + PIN external session) as a second factor before
signing. With it set, a link-only signing token alone is rejected
`STEP_UP_REQUIRED` at `.sign` (see below); the flag is surfaced on `.view` so the
portal knows whether to show the OTP step.

### `signature_request.send` (member)

Transitions `draft|sent → sent`, mints the per-request unguessable signing-link
token, and emails ONE branded invitation via the `EmailSink`.

```jsonc
{
  "request_id": "sigreq_…",                                   // required
  "signing_url_base": "https://apps.canonify.app/<org>/<app>", // optional but PRIMARY
  "document_title": "Mutual NDA",                              // optional, email body
  "sender_name": "Acme Legal"                                 // optional, email body
}
```

Pass `signing_url_base` (the signing-portal serving base) so the email renders a
"Review & Sign Document" button to the full portal URL and the raw token is
never shown; without it the email falls back to raw-token instructions. A
request not in `draft|sent` (e.g. already `signed`) → typed `CONFLICT`.

### `signature_request.view` (external, read)

Redeems the token and returns the UNSEALED document PDF as a base64 data-URL
plus the request fields. A `sent` request transitions `sent → viewed` on open;
a `signed` request also returns the sealed PAdES receipt.

```jsonc
{
  "request_id": "sigreq_…",       // required
  "link_token": "<emailed token>" // required (redacted from the audit trail)
}
```

Viewable from `sent | viewed | signed`. Rejects non-external principals, bad
tokens, a wrong bound actor, an expired link, and a not-yet-sent / `declined`
request with typed errors. The returned `require_step_up` tells the portal
whether to show the OTP step.

### `signature_request.sign` (external)

The crux: redeem the token, write the IMMUTABLE `signature` record
(`record_hash`-bound over the evidence fields), PAdES-seal the document (the
original contract + an appended Certificate of Completion, sealed once with the
per-org self-signed cert, SubFilter `ETSI.CAdES.detached`), and transition the
request → terminal `signed`.

```jsonc
{
  "request_id": "sigreq_…",                       // required
  "link_token": "<emailed token>",                // required (redacted at rest)
  "signer_name": "Sam Signer",                    // required, recorded onto the signature
  "signature_image": "data:image/png;base64,…",   // required; data-URL or bare base64 (redacted at rest)
  "consent": {                                     // required — explicit intent affirmation
    "acknowledged": true,                          // MUST be true, else VALIDATION
    "consent_version": "v1"                         // MUST name a known CONSENT_STATEMENTS entry
  }
}
```

**Consent is mandatory.** `consent.acknowledged` must be `true` and
`consent_version` must name a known entry in the server-side `CONSENT_STATEMENTS`
map (`v1` today). The action resolves the canonical `consent_text` from that map
server-side (it never trusts a client-sent statement) and binds both the text
and the version into `record_hash`. An unknown version → typed `VALIDATION`.
Signable only from `sent | viewed`. If the request was created with
`require_step_up`, a link-only signing token (server-derived `auth_method ===
'link_token'`) is rejected `FORBIDDEN/STEP_UP_REQUIRED` — only the OTP/session
path (`magic_link_pin`) may sign. One signature per request is structurally
enforced (a unique index on `_signature(request_id)`); a re-sign → `CONFLICT`.

### `signature_request.decline` (external)

The bound signer redeems their token and DECLINES instead of signing. Same auth
as `.sign`.

```jsonc
{
  "request_id": "sigreq_…",          // required
  "link_token": "<emailed token>",   // required (redacted at rest)
  "decline_reason": "Wrong counterparty" // optional free text — recorded in the audit args
}
```

A **standalone** request goes terminal `declined` (frozen). An **envelope slot**
goes `declined`, **its envelope goes `declined`, and every other still-open
sibling slot is swept `voided`** — declining any party ends the whole envelope.
Idempotent on an already-`declined` request (`{ already_declined: true }`).
There is no `decline_reason` column — the reason is recorded durably via the
action's audit/outbox args (the `link_token` is still redacted from them).

---

## `signature` — verifying the signed record

### `signature.verify` (authorized member, read-only)

Proves an existing signature is untampered + identity-bound. Re-derives the
`record_hash` over the stored evidence, re-derives the document content hash from
the unsealed PDF, and validates the PAdES seal (SubFilter present + the embedded
signing cert matches the org's cert).

```jsonc
{ "signature_id": "sig_…" }   // required
```

Returns `{ valid, reasons }` where `valid === (reasons.length === 0)` and each
reason is a stable string (`"record_hash mismatch"`,
`"document content hash mismatch"`, `"seal invalid: …"`). An **unknown
`signature_id`, or a signature the calling member is not authorized to read**
(its `signature_request` row is filtered out by the member floor
`member_account_id = :principal.id`), returns the **identical typed
`NOT_FOUND`** — a different answer from "invalid", and deliberately
indistinguishable between the two cases so `verify` can't be used as an
existence oracle. An admin/owner or the designated member party verifies
normally. `verify` has no external-signer branch.

### `signature.certificate` (authorized member or matching signer, read)

Returns the Certificate of Completion PDF as a base64 data-URL.

```jsonc
{ "signature_id": "sig_…" }   // required
```

Returns `{ signature_id, certificate_data_url, source }` where `source` is
`"sealed"` (the PAdES-sealed combined PDF the sign flow produced) or
`"rendered"` (a fresh certificate render, the fallback for a legacy row with no
sealed blob). Scope: an org member **that can read the signature's underlying
`signature_request` row** (routed through the same PolicyService read gate the
sibling reads use — a restricted member floored to
`member_account_id = :principal.id` that cannot read the request gets
`FORBIDDEN`; an admin/owner or the designated member party succeeds) — OR, on
the external proxy, the signer whose `external_actor_id` matches the
signature's `signer_actor_id`. Unknown id → `NOT_FOUND`.

---

## `signature_envelope` — the ordered multi-signer lifecycle

Status enum (`packages/canon/src/object-types/signature-envelope.ts`):

```
draft → in_progress → completed
                   ↘ declined
                   ↘ voided
```

`signature_envelope.create` lands `in_progress` directly (the `draft` envelope
state is reserved/unused). `completed` is the all-signed success terminal;
`declined` and `voided` are the abort terminals. All three are TERMINAL — turn
advance and completion never run on an aborted envelope (a defense-in-depth
guard beyond the per-slot status gate). The envelope's slots are themselves
`signature_request` rows (`envelope_id` set, `order_index` = signing position).

### `signature_envelope.create` (member)

```jsonc
{
  "document_content_hash": "<64-hex>",   // REQUIRED by the governance gate
  "document_file_id": "file_…",          // REQUIRED by the governance gate
  "title": "Mutual NDA",                  // optional
  "signers": [                            // ordered; array position = signing order; ≥1
    { "party_label": "Buyer",  "channel": "external", "email": "buyer@example.com" },
    { "party_label": "Seller", "channel": "member",   "member_account_id": "acct_…" }
  ]
}
```

Both document fields are required via the gate (schema-optional, gate-enforced —
mirroring `signature_request.create`). Each signer needs a non-blank
`party_label` and a channel-keyed identity: `channel: "external"` requires
`email`; `channel: "member"` requires `member_account_id`. Lands the envelope
`in_progress` with one slot per signer — slot 0 active (`draft`), every later
slot `pending`. Does **not** mint tokens or send — that is `.send`.

### `signature_envelope.send` (member)

Opens the envelope's **current turn** (the lowest-`order_index` unsigned slot)
and persists the per-envelope invitation defaults reused as the envelope
advances.

```jsonc
{
  "envelope_id": "sigenv_…",                                   // required
  "signing_url_base": "https://apps.canonify.app/<org>/<app>", // optional; persisted + reused per slot
  "sender_name": "Acme Legal",                                 // optional; persisted
  "document_title": "Mutual NDA"                               // optional; defaults to the envelope title
}
```

An **external** current slot gets its clickable invitation minted + emailed; a
**member** current slot is left active **without** an email (the dashboard
countersign owns member delivery). Sequential: one slot opens at a time. A
non-`draft|in_progress` envelope → typed `CONFLICT`.

### `signature_envelope.countersign` (member)

The member counterpart to the external `signature_request.sign`: an org member
countersigns their `channel='member'` slot from the dashboard. No link, no
`link_token` — the authenticated member session is the authorization.

```jsonc
{
  "request_id": "sigreq_…",                       // required — the member SLOT to countersign
  "signer_name": "Sam Supplier",                  // required
  "signature_image": "data:image/png;base64,…",   // required (redacted at rest)
  "consent": { "acknowledged": true, "consent_version": "v1" } // required, same contract as .sign
}
```

Writes the immutable `signature` row with the **same** evidence shape an external
slot records — except `auth_method` is the fixed `'member_session'` and
`signer_actor_id` is `member:<account_id>`. Per-slot sealing is **not** done
here; the final combined PAdES seal is the completion step. Guards: member-only
kind; the slot must be a `member` slot assigned to the acting member
(`member_account_id === principal.id`); the envelope turn gate (no earlier
unsigned sibling, else `CONFLICT/NOT_YOUR_TURN`); the slot must be active
(`draft`). On success it transitions the slot → `signed` and advances the
envelope (opens the next slot, or triggers completion when it was the last).

### Completion is automatic — there is no `signature_envelope.complete` action

When the **last** slot signs (external sign or member countersign), the
sign/countersign handler runs the completion step **in-transaction** —
internally, not as a separately-invoked action. It assembles the final PDF
(original document + a multi-signer Certificate of Completion), PAdES-seals it
ONCE, stores it as the envelope's `completed_file_id`, flips the envelope →
`completed`, repoints every slot signature's seal at the completed PDF, and
emails all parties a download link. It is idempotent. You do not call it; you
observe the envelope reaching `completed` and read the result via `.download`.

### `signature_envelope.download` (party only, read)

Returns the COMPLETED envelope's final sealed PDF as a base64 data-URL.

```jsonc
{ "envelope_id": "sigenv_…" }   // required
```

Returns `{ envelope_id, status, completed_file_id, document_data_url }`.
**Strictly party-scoped**: an external signer whose `external_actor_id` matches a
signed slot, or the member whose account id matches a member slot. A non-party
(member or external) and any other kind → `FORBIDDEN`. An unknown envelope →
`NOT_FOUND`; one not yet `completed` → typed `CONFLICT/NOT_COMPLETED`.

### `signature_envelope.my_pending_countersigns` (member, read)

The dashboard queue: the slots awaiting the **acting member's** countersign.

```jsonc
{}   // no args — the queue is resolved entirely from the member principal
```

Returns `{ pending: [{ request_id, envelope_id, party_label, order_index, title,
document_content_hash, document_file_id }] }` — exactly the member's
`channel='member'`, `member_account_id = self`, active (`draft`, current-turn)
slots. Strictly self-scoped server-side (the server binds `member_account_id`
from `principal.id`), so a member can never see another member's queue.

### `signature_envelope.decline` (member)

The member counterpart to `signature_request.decline`: an org member declines
their active member slot from the dashboard (member-session auth, no token).

```jsonc
{
  "request_id": "sigreq_…",              // required — the member SLOT to decline
  "decline_reason": "Terms unacceptable" // optional free text — recorded in the audit args
}
```

Same propagation as the external decline: the slot → `declined`, **the envelope
→ `declined`, and every other still-open sibling slot → `voided`**. Idempotent
on an already-declined slot. Guards mirror `.countersign` (member-only; the slot
must be a member slot assigned to the caller; turn gate; active `draft`).

### `signature_envelope.void` (member, sender side)

The sender abort: an org member voids an in-flight (`draft | in_progress`)
envelope.

```jsonc
{
  "envelope_id": "sigenv_…",       // required
  "void_reason": "Superseded"      // optional free text — recorded in the audit args
}
```

The envelope goes terminal `voided` and **every still-open slot
(`pending|draft|sent|viewed`) is swept `voided`** — clearing the countersign
queue so no party can sign afterwards. Idempotent (`{ already_voided: true }`).
A `completed` or already-`declined` envelope → `CONFLICT`. There is no
creator/sender account column on the envelope, so **any** org member may void;
org-scoping is the tenancy boundary.

---

## `member_signature` — the reusable saved signature

A DocuSign-style account-holder signature a member saves once and reuses on every
dashboard countersign (no redraw). This is a **convenience store**, NOT a
tamper-evident record — it carries no consent / evidence / `record_hash`; the
real signing evidence is written only at countersign time onto `_signature`.

### `member_signature.set` (member)

```jsonc
{
  "signer_name": "Sam Supplier",                  // required
  "signature_image": "data:image/png;base64,…",   // required; data-URL or bare base64 PNG (redacted at rest)
  "source": "typed"                                // required: "typed" | "drawn"
}
```

Stores the client-produced PNG bytes in the FileStore and upserts a per-member
row keyed to `(org, account)`. Set-once then updatable (preserves the original
`created_at`, bumps `updated_at`). `source` is `typed` (a name rendered to PNG)
or `drawn` (a signature_pad capture) — audit/UX metadata. Returns
`{ account_id, signer_name, source, signature_image_content_hash, created_at,
updated_at }` (the FileStore id is deliberately not surfaced — the content hash
is the stable cross-surface identity).

### `member_signature.get` (member, read)

```jsonc
{}   // no args — resolved from the acting member's principal
```

Returns `{ has_signature, signer_name, source, signature_image,
signature_image_content_hash, created_at, updated_at }` — the saved image as a
base64 data-URL for preview + one-tick countersign reuse, or
`has_signature: false` (all fields null) when none is set yet. Strictly
self-scoped — a member can only read their own.

---

## Source-of-truth references

- [`packages/canon/src/actions/signature-request/`](../../../../packages/canon/src/actions/signature-request/) — `create` / `send` / `view` / `sign` / `decline` inputs + guards.
- [`packages/canon/src/actions/signature/`](../../../../packages/canon/src/actions/signature/) — `verify` / `certificate`.
- [`packages/canon/src/actions/signature-envelope/`](../../../../packages/canon/src/actions/signature-envelope/) — `create` / `send` / `countersign` / `download` / `my-pending-countersigns` / `decline` / `void`, and `complete.ts` (the automatic completion step).
- [`packages/canon/src/actions/member-signature/`](../../../../packages/canon/src/actions/member-signature/) — `set` / `get`.
- [`packages/canon/src/object-types/signature-request.ts`](../../../../packages/canon/src/object-types/signature-request.ts) / [`signature-envelope.ts`](../../../../packages/canon/src/object-types/signature-envelope.ts) — the status enums + `parts_frozen_when`.
- [`packages/canon/src/types.ts`](../../../../packages/canon/src/types.ts) — `RequestContext` (the server-derived evidence seam).
- [`repo-wiki/operations/signing-protocol.md`](../../../../repo-wiki/operations/signing-protocol.md) — the browser-portal surface: the external `/_sign` page, magic-link + PIN session, and the `/_canon/sign-token` + `/_canon/download-token` proxy endpoints.
