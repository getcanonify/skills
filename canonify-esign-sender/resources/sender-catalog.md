# Sender catalog — layout

The `sender-catalog/` directory next to this file is the bundle the skill
applies (what each artifact does, the read fence and the apply command are in
the skill's `SKILL.md`).

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

`object-types/`, `schema/`, `actions/`, `state-machines/`, `forms/` and `apps/`
are absent on purpose — the object type and actions are already in every
canon, and `canon catalog apply` tolerates missing section directories.
