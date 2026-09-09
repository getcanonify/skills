---
name: canonify-esign-sender
description: Reference ViewSpec catalog for the Canonify e-signature SENDER surface — a creator-scoped sent-agreements table (title/status/decline_reason + per-row void) and a create form — shipped as an on-disk catalog.apply bundle (views + policies only) that composes the platform signature_envelope object-type + create/void actions. Nothing is baked into the empty canon; apply on demand with `canon catalog apply`.
version: 1.0.0
---

# Canonify — E-signature Sender surface

This skill ships a **reference ViewSpec catalog** — not a new engine. It is the
Canon-customer **SENDER** surface for multi-signer signing envelopes:

- a **sent-agreements table** over `signature_envelope`, scoped to the envelopes
  the acting member CREATED, showing `title`, `status`, and `decline_reason`,
  with a per-row **Void** action; and
- a **create form** (a `panel` bound to `signature_envelope.create`) for sending
  a new agreement.

It composes surfaces that **already exist in every canon**: the platform
`signature_envelope` object-type and the `signature_envelope.create` /
`signature_envelope.void` ActionTypes (registered unconditionally by the canon
engine — see `packages/canon/src/catalog.ts`). The bundle carries only **views**
and **policies** — its `object-types/`, `schema/`, `actions/`, `state-machines/`,
`forms/`, and `apps/` sections are empty. **Nothing is baked into the empty
canon**; this catalog is the on-demand, skill-distributed artifact.

## Prerequisite — `creator_account_id` + the server-side read-fence (x20)

`signature_envelope.create` stamps `creator_account_id` = the acting principal's
account id. The canon's in-code member policy floor scopes every member read of
`signature_envelope` to `creator_account_id = :principal.id`
(`policy-service.ts`, epic 2hg / ADR 0017 §6, tick x20). So the table's
`source.filter: { creator_account_id: '$principal' }` is **defense-in-depth /
documentation** of a fence the server already enforces — a member can never read
another member's envelopes even if the ViewSpec filter were removed. The read
**policy** in this bundle documents and grants the read; the fence does the
row-scoping. This requires a canon at or beyond x20 (the `creator_account_id`
property must be exposed on `signature_envelope`).

## What the bundle contains

`resources/sender-catalog/` is an on-disk bundle in **`catalog.export` layout**
(a `views/` dir + a `policies/` dir; every other section empty), ready for
`catalog.apply`:

```
resources/sender-catalog/
  manifest.json                 # bundle_version + per-kind counts (2 views, 3 policies)
  views/
    SignatureEnvelope.sent.json   # table: creator-scoped list + void rowAction
    SignatureEnvelope.create.json # panel: actionForm bound to signature_envelope.create
  policies/
    role-member__object_type-signature_envelope__get-list.json      # member read
    role-member__action-signature_envelope.create__invoke.json      # member invoke create
    role-member__action-signature_envelope.void__invoke.json        # member invoke void
```

- **`SignatureEnvelope.sent`** — a `table` ViewSpec over `signature_envelope`
  with `source.filter: { creator_account_id: '$principal' }`, columns `title` +
  `status` + `decline_reason`, and a `rowActions` entry
  `{ action: 'signature_envelope.void', presetParams: { envelope_id: '$row.id' } }`
  (ADR 0018 P3 curated per-row action; `$row.id` is resolved at render time).
- **`SignatureEnvelope.create`** — a `panel` ViewSpec whose `actionForm` binds
  `signature_envelope.create` (ADR 0018 §2). The form fields (document,
  signers, …) are derived from that action's input schema.
- **policies** — the org `member` role gets `get`/`list` on `signature_envelope`
  (the read the table needs; the x20 fence still scopes it server-side) and
  `invoke` on `signature_envelope.create` and `signature_envelope.void`.

The bundle carries **no `managed_by` provenance intent** of its own — apply
stamps every artifact `managed` via the privileged provenance seam server-side
(the `managed_by`/`source_ref` keys on each on-disk record are required by
`CatalogBundleSchema` to decode, but are inert on apply; the server never trusts
them). Do not hand-edit them expecting them to change provenance.

## Locale-aware sending

`signature_envelope.create`'s input carries two independently-optional locale
fields (the closed `en` / `da` set). Because `SignatureEnvelope.create`'s form
fields are derived from the action's live input schema (see above), both
already appear on the form without any bundle change — there's nothing to add
to `SignatureEnvelope.create.json`.

- **`locale`** (envelope-level) — the **document's** language: the completed
  contract PDF and the signing certificate render in it. Envelope-level, not
  per-signer, because the document is rendered once and is the same artifact
  for every signer regardless of who they are.
- **`signers[].locale`** (per-signer) — the **party's** language: their
  invitation email and signing-portal session render in it. Per-signer, so one
  envelope can pair a Danish customer with an English counterparty off the
  same document, each invited in their own language.

The two locale fields fall back independently when a signer states none:

- the **invitation email** falls back to the envelope's `locale` (an email has
  no request to read an `Accept-Language` header from), then the platform
  default;
- the **signing portal**, once the signer opens the link, falls back to their
  browser's `Accept-Language`, then the platform default — so a signer who
  never stated a locale can still get a portal in their own browser's language
  even though their invitation email rendered in the envelope's.

**Sender notifications matter here too.** The email the creator receives when
an envelope completes or is declined is sent in the **envelope's** locale, not
the sender's own profile preference — the canon has no reach into the
control-plane's per-user locale setting, so the envelope's declared language is
the best signal available. A member who creates an envelope with `locale:
'da'` gets its completion/decline notice in Danish even if their own UI is set
to English.

## How to apply it

Install the skill, then apply the vendored bundle into your canon:

```sh
# read the canon's current baseline token (optimistic-concurrency guard)
canon catalog export /tmp/current --managed        # prints the baseline token on stdout

# apply the sender catalog, pinning that baseline (or --force to overwrite)
canon catalog apply resources/sender-catalog --baseline <token>
```

- The guard is **default-on**: pass either `--baseline <token>` or `--force`
  (a bare apply is refused `BASELINE_REQUIRED`). When you pin a baseline, apply
  is rejected `409 STALE_CANON` before any write if any artifact it would change
  or prune has drifted since — re-read and rebuild. This hand-authored bundle
  carries no live baseline of its own (the token is per-canon LIVE state, never
  part of the diffable source), so read it from `catalog.export`'s stdout, not
  from this bundle's `manifest.json`.
- Apply is **idempotent**: re-applying the same bundle is a no-op (views UPSERT
  by name; policies de-dupe by content). Every artifact lands `managed`.
- Apply is gated on the `manage` operation — run it as owner/admin, not a member.

## Adapting it

These are reference specs — adapt names, columns, or add a `detail` view. Any
edit must still decode against the canon ViewSpec schema
(`packages/canon/src/view-spec/schema.ts`) and the `CatalogBundleSchema`
(`packages/canon/src/catalog-bundle.ts`); a `catalog.apply` re-validates every
view against the live registry (real ObjectType, real columns, real actions).
