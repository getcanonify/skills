# Object property visibility

Two separate mechanisms control what a caller sees: the per-property `hidden`
flag (refinable from an agent surface) and the per-ObjectType `confidential`
marker (also refinable, but at ObjectType scope). They answer different
questions. This page is verified
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

`confidential` is a boolean on the **ObjectType**, not a property, and is
authorable in the same `catalog.refine_object_type` call as the rest of the
ObjectType declaration. Set it to `true` when an app's agent declares that the
type contains sensitive data; you can also read the effective value via `canon
schema describe --object <name>`.

When an ObjectType is `confidential: true` (Phase 28.5):

1. **Default-deny without a grant.** Under the production deny-by-default
   posture the type denies reads/invokes unless an explicit grant exists —
   "confidential" is simply "no grant exists." A blocked access writes its
   audit row with `status='denied'`.
2. **Granted mutations are flagged in the audit trail.** A granted invoke that
   touches a confidential type records `status='sensitive'` instead of
   `'success'` (`runtime.ts`). Granted reads use the reads-denials-only policy
   (perf decision 2026-07-07): they persist no row at all and the envelope's
   `audit_event_id` is empty (`''`), so only *denied* attempts are queryable:

   ```sh
   canon audit --status denied
   ```

   A granted **write** invoke that touches a confidential type is still fully
   audited, with `status='sensitive'` instead of `'success'` (`runtime.ts`),
   so `canon audit --status sensitive` lists granted confidential
   *mutations* — it does **not** answer "who *read* the confidential data";
   there is no persisted trail of granted reads.

ObjectTypes that omit `confidential` keep emitting ordinary `'read'` /
`'success'` statuses, so the marker introduces no audit noise.

## Which one do I want?

- "Keep this column out of the default shape but a handler may still use it" →
  property `hidden`.
- "This whole type holds sensitive data; keep the production default-deny
  posture and flag granted mutations in the audit trail" → ObjectType
  `confidential: true` in `catalog.refine_object_type` (decision 0033). Note
  granted *reads* leave no trail (denials-only audit, 2026-07-07).

## See also

- [object-property-types.md](object-property-types.md) — the `mapped` property shape `hidden` lives on.
- Base skill: `canon skill --name canonify resource declaring-objecttypes.md` and `canon audit` in the base SKILL.md.
