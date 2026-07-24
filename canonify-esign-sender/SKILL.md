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

## How to apply it

Install the skill, then apply the vendored bundle into your canon:

```sh
# read the canon's current revision (optimistic-concurrency guard)
canon catalog export /tmp/current --managed        # manifest.revision is stamped here

# apply the sender catalog (optionally pin --expected-revision <n> from above)
canon catalog apply resources/sender-catalog --expected-revision <n>
```

- `--expected-revision <n>` is **optional**. When supplied it must equal the
  canon's current revision or apply is rejected `409 STALE_REVISION` before any
  write — re-read and rebuild. This hand-authored bundle carries no live
  revision of its own (revision is per-canon LIVE state, never part of the
  diffable source), so read it from `catalog.export`'s manifest, not from this
  bundle's `manifest.json`.
- Apply is **idempotent**: re-applying the same bundle is a no-op (views UPSERT
  by name; policies de-dupe by content). Every artifact lands `managed`.
- Apply is gated on the `manage` operation — run it as owner/admin, not a member.

## Adapting it

These are reference specs — adapt names, columns, or add a `detail` view. Any
edit must still decode against the canon ViewSpec schema
(`packages/canon/src/view-spec/schema.ts`) and the `CatalogBundleSchema`
(`packages/canon/src/catalog-bundle.ts`); a `catalog.apply` re-validates every
view against the live registry (real ObjectType, real columns, real actions).
