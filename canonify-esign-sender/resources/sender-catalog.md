# Sender catalog — apply guide

The `sender-catalog/` directory next to this file is an on-disk **`catalog.apply`
bundle** in `catalog.export` layout. It ships the Canon-customer e-signature
**SENDER** surface as reference ViewSpecs + policies. It bakes **nothing** into
the empty canon — it composes the platform `signature_envelope` object-type and
the `signature_envelope.create` / `.void` actions that every canon already
registers.

## Layout

```
sender-catalog/
  manifest.json                 # bundle_version + counts (views: 2, policies: 3; all else 0)
  views/
    SignatureEnvelope.sent.json
    SignatureEnvelope.create.json
  policies/
    role-member__object_type-signature_envelope__get-list.json
    role-member__action-signature_envelope.create__invoke.json
    role-member__action-signature_envelope.void__invoke.json
```

`object-types/`, `schema/`, `actions/`, `state-machines/`, `forms/`, and `apps/`
are intentionally **absent** — the engine (object-type + actions) is already in
every canon. `catalog apply`'s `readBundle` tolerates missing section dirs.

## Views

- **`SignatureEnvelope.sent`** (`table`) — over `signature_envelope`, filtered
  `creator_account_id: '$principal'`, columns `title` / `status` /
  `decline_reason`, with a per-row **Void** action
  (`signature_envelope.void`, `presetParams.envelope_id: '$row.id'`).
- **`SignatureEnvelope.create`** (`panel`) — an `actionForm` bound to
  `signature_envelope.create`. The form fields come from that action's input
  schema (document + ordered signer slots), including the optional
  envelope-level `locale` and per-signer `signers[].locale` fields — see
  "Locale-aware sending" in `SKILL.md` for what each one controls.

## Policies

Grant the org `member` role:

- `get` + `list` on `signature_envelope` (the read the table needs);
- `invoke` on `signature_envelope.create`;
- `invoke` on `signature_envelope.void`.

## The read-fence (x20)

Member reads of `signature_envelope` are scoped **server-side** to
`creator_account_id = :principal.id` by the in-code member policy floor
(`policy-service.ts`, epic 2hg / ADR 0017 §6). The `get`/`list` policy in this
bundle documents and grants the read; the fence enforces the row-scope. The
ViewSpec's `source.filter` mirrors the fence for clarity — it is not the security
boundary.

## Applying

```sh
canon catalog export /tmp/current --managed          # prints the baseline token on stdout
canon catalog apply sender-catalog --baseline <token>
```

The guard is default-on: pass `--baseline <token>` or `--force`. When you pin a
baseline it is checked per-artifact — apply is rejected `409 STALE_CANON` if any
artifact it would change or prune has drifted since (read the token from
`catalog.export`'s stdout — this bundle's `manifest.json` carries no live
baseline). Re-applying is idempotent (views UPSERT by name, policies de-dupe by
content); every artifact is stamped `managed` provenance server-side. Apply
requires owner/admin (`manage`), not a member.
