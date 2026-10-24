---
title: The App record and the catalog bundle — export, validate, apply, scratch
description: Group views into an App (nav, defaultView, Browse data, count badges), then crystallize the canon as a version-controlled bundle and reconcile orgs to it with the total validator, pruning apply, and the per-artifact baseline guard.
---

# The App record and the catalog bundle

## The App record — `apps.declare`

An App is a named, ordered grouping of declared Views; it owns nothing.

```sh
canon actions invoke apps.declare --input '{
  "slug": "crm", "name": "CRM", "icon": "briefcase",
  "nav": [
    { "section": "Pipeline", "view": "Deal.pipeline" },
    { "section": "Pipeline", "view": "Customer.list",
      "count": { "objectType": "customer", "filter": { "status": "active" } } }
  ],
  "defaultView": "Deal.pipeline"
}'
```

- A `nav` entry is exactly `{ section?, view, count? }` — an `icon` or `label`
  on it is refused (`invalid_artifact`); icons live on the App or the View.
- `defaultView` — the nav view the app opens on instead of the tile overview.
  It must be one of the app's own nav views (an out-of-nav value is ignored),
  and it **replaces** the Home nav entry, so the page appears in the nav once.
  A caller who may not open it lands on the first nav view they may (limiting
  a view or app to a group: `canonify-objects-actions` access-policies.md).
- `count` — a live row-count badge on a nav item: `{ objectType, filter? }`.
- `browse` — an explicit list of ObjectType names for the "Browse data"
  sidebar. Omitted (the default), the list is **derived** from the types the nav
  views target, each expanded to its whole `part_of` compound, so a curated app
  does not list every org table. Unknown names are dropped at render;
  re-declaring without `browse` clears an override.
- Re-declaring an **existing** slug needs `expectedHash` (from
  `catalog.export`) or `force: true`; a bare re-declare is
  `CONFLICT/HASH_REQUIRED`, a mismatch `CONFLICT/STALE_DECLARATION`. The first
  declare of a fresh slug has nothing to conflict with.
- `supported_locales` / `default_locale`: see [bilingual-apps.md](bilingual-apps.md).

## The bundle

```sh
canon catalog export ./my-app-canon       # writes the bundle; prints a JSON summary
```

A bundle is a directory of declarative JSON, one file per artifact
(`schema/`, `object-types/`, `actions/`, `fragments/`, `state-machines/`,
`views/`, `apps/`, `groups/`, `policies/`, `forms/`, `i18n/`, `email-templates/`, plus a
`manifest.json` index; missing sections are tolerated). It is the source of
truth: grow the canon with the verbs, export, commit, review per-artifact diffs.
Export → apply to a fresh org is byte-identical (a conformance gate), so keep
the committed bundle and the live org in sync.

| Verb | Does | Live org? |
|---|---|---|
| `canon catalog export <dir> [--managed] [--baseline-only]` | Write the catalog as a bundle; print a JSON summary whose `baseline` field is the token (`--baseline-only`: just the token) | reads |
| `canon catalog validate <dir> [--strict] [--target\|--live <env>]` | Every diagnostic `apply` would raise, in one read-only pass; an error caused by an artifact refused above carries `depends_on` and prints as one note under it — fix the root first | no (unless `--target`/`--live`) |
| `canon catalog audit <dir>` | The coherence rules (`C1-shadow-entity`, `C10-float-money`, `C11-wildcard-grant`) plus every validate diagnostic | no |
| `canon catalog test <bundle-dir> [<scenarios-dir>]` | Boot an empty in-process canon, apply, run every `session.yaml`, tear down — see [`canonify-scenarios`](../../canonify-scenarios/SKILL.md) | no |
| `canon catalog scratch <invocations.json> <export-dir>` | Replay organic declarations into an empty in-process canon and export the bundle | no |
| `canon catalog apply <dir> --baseline <token> \| --force` | Reconcile the org's managed partition to the bundle | writes |

Evolve it either way: declare organically and re-export (per-step validation,
best for exploring), or edit the JSON and apply (best for bulk, mechanical
changes — apply re-validates, so a sloppy edit fails instead of corrupting).

## `validate` and `apply` — a total validator

Apply replays every declaration through the same validating handlers as the
organic verbs and cross-checks the links between artifacts. A bundle it accepts
renders and runs with no definition-level error, and
**validate-rejects ≡ apply-rejects** — including for a greenfield bundle: the
CLI `validate` boots a throwaway canon, materializes the bundle's schema and
ObjectTypes, and runs every check against them (the REST/MCP `catalog.validate`
action runs the same gate on a detached copy of the live registry). It reports
every problem in one pass and exits non-zero, so use it as the CI pre-flight.
The interactive verbs (`view_specs.declare`, `catalog.refine_object_type`,
`actions.update`, `apps.declare`, `groups.declare`, …) run the same checks and
refuse with the same code; a warning comes back in the result's `advisories`.
Warnings (`unresolvable_title`, `missing_view_icon`, …) fail only under
`--strict` (`strict: true`).

| `code` | Means |
|---|---|
| `invalid_artifact` | An artifact fails its closed declaration schema — an unknown key is refused, not ignored. |
| `unresolvable_subject` | A table `rowActions` / `queue` / `board` action has no resolvable subject ([the rule](../../canonify-objects-actions/resources/actions-input-governance.md#subject)): no `<type>_id` input, no single required input referencing the row type, no declared `subject`. |
| `unresolved_action_subject` (warning) | An action takes a required id input yet resolves no subject, so no record offers it; the message names the `subject` to declare. |
| `unresolvable_ref` | A queue `source.objectType`, `order[].field`, `urgency.from` (must be an enum), `$field` / `$link.property` / `{{age <field>}}` token, `row.peek` or table `peek` does not resolve (another type or block is `invalid_queue_row` / `invalid_table_peek`). |
| `unknown_column_ref`, `source_object_type_mismatch`, `invalid_board_group_by`, `unknown_action`, `duplicate_block_heading`, … | Every `view_specs.declare` check, replayed — fixes on the [cheatsheet](../../canonify-viewspecs/resources/viewspec-cheatsheet.md). |
| `registry_content_mismatch` | A handler addresses a property whose live type differs from the bundle's (apply never ALTERs a live table). It leads; the handler type errors it causes follow in `details.diagnostics`. Only addressed properties are compared, and a property the bundle re-declares is excluded. |

A refused apply is refused **before the first write**: `VALIDATION` with
`details.apply_state: "pre_write"` and every blocking violation in
`details.diagnostics`; the org is untouched. Past that pre-flight, atomicity is
per sub-step: a failure the pre-flight cannot model (a row check against live
data, a DDL step) leaves earlier steps committed, says so with
`details.apply_state: "partial_durable"`, and a clean re-apply converges.

**Apply never changes a live table.** `schema/` only bootstraps tables that do
not exist (each entry is a full `CREATE TABLE`; "already exists" counts as
success); a differing live DDL raises the advisory `schema_ddl_divergence`. A
property on a **new column of an existing table** is two steps:

```sh
canon actions invoke schema.propose_migration --input '{"description":"staff_member.employment_type","sql":"ALTER TABLE staff_member ADD COLUMN employment_type TEXT"}'
canon catalog apply ./my-app-canon --baseline <token>
```

Without the migration, apply refuses: `Property "employment_type" maps to column
"employment_type" which does not exist on table "staff_member"`. To store a
value a derived property computes, add the column under the property's name the
same way: the migration turns the property mapped, and the bundle then declares
it mapped ([derived properties](../../canonify-objects-actions/resources/object-property-types.md#derived-properties)).

## Apply prunes

Apply reconciles the **managed** partition to exactly the bundle: a managed
artifact dropped from the bundle is **deleted** (apps → forms → policies → views
→ state machines → actions → object types). Artifacts grown organically in the
org (`tenant` partition) are never pruned. Removing one **property** from a
refinement does not drop its column — auto-CRUD re-infers it, so the org drifts
from a fresh one; apply warns `removed_property_not_dropped` (never blocking,
even under `--strict`) and points at a `DROP COLUMN` migration.

## The baseline guard

Every apply against a canon that already holds declarations says which it
means:

```sh
T=$(canon catalog export /tmp/canon-live --baseline-only)   # fresh export + its token
canon catalog plan ./my-app-canon                           # what the apply would add, change, prune
canon catalog apply ./my-app-canon --baseline "$T"          # apply only if nothing I change has drifted
canon catalog apply ./my-app-canon --force                  # deliberate overwrite
```

The first apply to an empty org (no tables, artifacts, message files or
managed policies — the provisioned owner policy does not count) needs neither:
there is nothing to overwrite. The token (the `baseline` field of `export`'s
JSON summary, or the whole output with `--baseline-only`; never written into
`manifest.json`) fingerprints every artifact. For each artifact the apply would change or prune, a drift since
export rejects the apply before any write: `STALE_CANON` (409) with
`details.drifted` (`action:<name>`, `view:<name>`, …) — re-export, re-plan,
re-apply. Disjoint edits pass. Other refusals: neither flag →
`BASELINE_REQUIRED` (non-interactive callers fail closed; a human at a TTY gets a
`y/N` prompt); both flags → `CONTRADICTORY_BASELINE_INTENT`; an apply already in
flight → `APPLY_IN_PROGRESS` (retry); a baseline from a re-provisioned database →
`details.reason: 'epoch_mismatch'`. `--force` skips only the baseline check —
never declaration validation or `registry_content_mismatch`.

## `catalog scratch` — author without a live org

```sh
canon catalog scratch ./invocations.json ./my-app-canon
```

The file is `{ "invocations": [ { "action": "...", "input": {...} }, … ] }`,
replayed in order as the scratch canon's owner. A failing invocation aborts
naming the action; one that parks for approval aborts naming its `request_id`
(scratch cannot approve, and the bundle would silently lack it). Nothing durable
is left behind.
