---
name: canonify-esign-sender
description: Reference ViewSpec catalog for the Canonify e-signature SENDER surface — a creator-scoped sent-agreements table (title/status/decline_reason + per-row void) and a create form — shipped as an on-disk catalog.apply bundle (views + policies only) that composes the platform signature_envelope object-type + create/void actions. Nothing is baked into the empty canon; apply on demand with `canon catalog apply`.
version: 2.0.0
---

# Canonify — E-signature Sender surface

A **reference catalog**, not a new engine: the sender screens for multi-signer
signing envelopes, built on what every canon already registers — the platform
`signature_envelope` ObjectType and the `signature_envelope.create` /
`signature_envelope.void` actions (reference:
[../canonify-objects-actions/resources/esign-actions.md](../canonify-objects-actions/resources/esign-actions.md)).
Nothing is baked into an empty canon; you apply this bundle when an app
e-signs.

## What the bundle contains

`resources/sender-catalog/` is a `catalog.export`-layout bundle with only
`views/` and `policies/` (layout: [resources/sender-catalog.md](resources/sender-catalog.md)):

| Artifact | What |
|---|---|
| `SignatureEnvelope.sent` | `table` over `signature_envelope`, `source.filter: { creator_account_id: '$principal' }`, columns `title` / `status` / `decline_reason`, row action `signature_envelope.void` with `presetParams: { envelope_id: '$row.id' }` (redundant: the action declares subject `envelope_id`, so the row binds it; harmless) |
| `SignatureEnvelope.create` | `panel` whose `actionForm` binds `signature_envelope.create`; its fields (document, ordered signers, locales) come from the action's live input schema |
| 3 policies | role `member`: `get` + `list` on `signature_envelope`, `invoke` on `signature_envelope.create` and `signature_envelope.void` |

**The read fence is server-side.** `signature_envelope.create` stamps
`creator_account_id` with the acting member's account id, and the platform's
member policy scopes every member read of `signature_envelope` to
`creator_account_id = :principal.id`. The table's filter only mirrors that
fence; a member cannot read another member's envelopes even without it. The
read policy grants the read; the fence scopes the rows.

## Locale-aware sending

`signature_envelope.create` takes two independent, optional locales from the
closed `en` / `da` set; both appear on the form with no bundle change.

| Field | Controls | Fallback when unset |
|---|---|---|
| `locale` (envelope) | the document: completed PDF and signing certificate (rendered once, for every signer) | platform default |
| `signers[].locale` | that party's invitation email and signing-portal session | email: the envelope's `locale`, then platform default. Portal: the browser's `Accept-Language`, then platform default |

The creator's completion and decline notifications use the **envelope's**
locale, not the creator's own profile setting.

## Apply it

```sh
T=$(canon catalog export /tmp/current --managed --baseline-only)   # the canon's baseline token
canon catalog apply resources/sender-catalog --baseline "$T"       # or --force
```

Run as owner or admin (`manage`). The baseline is live canon state, so read it
from `catalog.export`, never from the bundle's `manifest.json`. Re-applying is
idempotent (views upsert by name, policies de-dupe by content) and every
artifact lands `managed`; the `managed_by` / `source_ref` keys in the files
are required by the bundle schema but ignored on apply. Baseline, `--force`
and `STALE_CANON` semantics:
[../canonify-app-authoring/resources/catalog-bundle.md](../canonify-app-authoring/resources/catalog-bundle.md).

## Adapting it

Rename views, change columns, add a `detail` view. Every edit must still decode
as a ViewSpec and a catalog bundle; `catalog.apply` re-validates each view
against the live registry (real ObjectType, columns and actions).
