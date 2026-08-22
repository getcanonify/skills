# Object property visibility

Two separate mechanisms control what a caller sees: the per-property `hidden`
flag (refinable from an agent surface) and the per-ObjectType `confidential`
marker (platform-tier). They answer different questions. This page is verified
against `MappedProperty.hidden` + `ObjectType.confidential` in
`packages/canon/src/types.ts`, the `ObjectPropertyRefinement` input schema in
`packages/canon/src/actions/catalog/refine-object-type.ts`, and the audit
status logic in `packages/canon/src/runtime-reads.ts` / `runtime.ts`.

## `hidden` — withhold (or surface) a column from the typed shape

`hidden` is an optional boolean on a `mapped` property. It is the one
visibility lever you set declaratively, through
`catalog.refine_object_type`'s `properties` map.

- A column the platform hid (e.g. `org_id`) can be **surfaced** so a declared
  handler may write through it — add the property with `hidden: false` (or
  omit `hidden` to make it ordinarily visible).
- A column you want present in the typed model but **kept out of the default
  read projection** is declared `hidden: true`.

```json
{
  "name": "customer",
  "properties": {
    "org_id": { "kind": "mapped", "column": "org_id", "type": "string", "hidden": false },
    "internal_notes": { "kind": "mapped", "column": "internal_notes", "type": "string", "hidden": true }
  }
}
```

```json
{
  "data": { "name": "customer", "refined": true, "refinement_id": "re_…" },
  "audit_event_id": "ae_…",
  "approval_status": "not_required",
  "knowledge_used": [],
  "outcome_signals": []
}
```

`hidden` is a property-shape toggle: it changes which columns the typed read
shape and the default detail/list pages expose. It is **not** an authorization
boundary on its own — a handler that explicitly references a hidden property by
name can still read/write it. Use it to keep noisy or plumbing columns
(`org_id`, audit bookkeeping) out of the agent-facing shape, not as a secret.

## `confidential` — the auth-context / audit marker

`confidential` is a boolean on the **ObjectType**, not a property, and it is a
**platform-tier marker** — it is deliberately absent from
`catalog.refine_object_type`'s input (which accepts only `name`,
`description`, `properties`, `links`, `defaults`). You cannot set it from an
agent surface; you read it via `canon schema describe --object <name>`.

When an ObjectType is `confidential: true` (Phase 28.5):

1. **Default-deny without a grant.** Under the production deny-by-default
   posture the type denies reads/invokes unless an explicit grant exists —
   "confidential" is simply "no grant exists." A blocked access writes its
   audit row with `status='denied'`.
2. **Granted access is flagged in the audit trail.** A *granted* read of a
   confidential type records `status='sensitive'` instead of `'read'`, and a
   granted invoke that touches one records `status='sensitive'` instead of
   `'success'` (`runtime-reads.ts` `auditStatusForRead`, `runtime.ts`). That
   makes "who accessed the confidential data" answerable directly:

   ```sh
   canon audit --status sensitive
   ```

   — symmetric with the `status='denied'` story for blocked access, without a
   reviewer having to cross-reference the catalog to learn which types are
   sensitive.

ObjectTypes that omit `confidential` keep emitting ordinary `'read'` /
`'success'` statuses, so the marker introduces no audit noise.

## Which one do I want?

- "Keep this column out of the default shape but a handler may still use it" →
  property `hidden`.
- "This whole type holds sensitive data; default-deny it and flag every
  granted touch in the audit trail" → ObjectType `confidential` (platform-tier;
  request it from the platform owner — not declarable from an agent surface).

## See also

- [object-property-types.md](object-property-types.md) — the `mapped` property shape `hidden` lives on.
- Base skill: `canon skill --name canonify resource declaring-objecttypes.md` and `canon audit` in the base SKILL.md.
