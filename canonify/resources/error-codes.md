# Error codes

Every failed call returns the same envelope on every surface, with a stable
`error.code` to switch on. Handlers raise domain failures as
`ActionDomainError(code, message, details?)`; the runtime rolls the transaction
back, writes the audit row and renders the error. Over REST the body is the full
envelope even on a 4xx — switch on `error.code`, never on the HTTP status.

```json
{
  "data": null,
  "audit_event_id": "aud_…",
  "knowledge_used": [],
  "outcome_signals": [],
  "error": {
    "code": "NOT_FOUND",
    "message": "lead not found (step 0: lead)",
    "issues": [],
    "details": { "code": "lookup_failed" }
  }
}
```

An error envelope carries no `approval_status` (a failed call settled no
approval question). `error` fields:

| Field | Meaning |
|---|---|
| `code` | The kind (table below), the stable wire contract to switch on. `message` is words for a person (mostly English; a platform refusal that carries `message_params`, and author copy, are in the org's language); localize on the client from `code` + `details`, falling back to `message`. |
| `issues` | Field-path issues; populated on `VALIDATION`. |
| `field_errors` | Optional `{ <field>: <message> }`. Filled on every input-decode failure (folded from `issues`) and whenever a handler's `details.field_errors` carries a map. Render inline on the field. |
| `failed_precondition` | Optional `{ field?, field_path? }`: the input an `assert` step with `else.field` was checking; its `message` is the author's copy. |
| `details` | Machine params: `details.code` (or `details.reason`) is the sub-code channel. A platform refusal a client may say in its own words also carries `message_key` (a `@t:` key) and the `message_params` its sentence is written from. |
| `suggestion` | Optional `knowledge.contribute` hint on a `FORBIDDEN` / `INVALID_STATE` denial. |

All of these are filled by the runtime before any surface sees the envelope, so
REST, CLI `--output json` and MCP carry the identical shape. Every text a
refusal shows (`message`, `field_errors`, a confirm, a warning) is words in the
org's language, never a raw `@t:` key: an author's keyed `else.message` or
confirm is resolved through the org's message bundles (its language, then
English, then the key's last segment as words), its placeholders filled. (The CLI's text
output additionally prints `VALIDATION` issues inline.)

## The kinds

| Code | CLI exit | HTTP | Fires when | Recover |
|---|---|---|---|---|
| `VALIDATION` | 2 | 400 | Input decode failed; a declaration failed its checks (`details.errors` names step and field for a handler type-check); a handler or generated write broke a database constraint; `direct_state_write`; `import_row_invalid`; `invalid_transition_step`. | `canon actions describe <name> --json` (or `--explain`) for the input schema; fix the named field. |
| `NOT_FOUND` | 3 | 404 | A `read` step matched nothing; the transition guard found no row; a foreign reference names no row; an unknown id, action or ObjectType. | Check the id (`canon objects get <type> <id>`), or `canon actions list`. |
| `FORBIDDEN` | 4 | 403 | Governance denied: principal kind not in `allowed_principals` (`details.reason: 'PRINCIPAL_KIND_NOT_ALLOWED'` with `{ principal_kind, action, allowed }`), a policy denial, or approval required. | Check governance in `actions describe`; switch identity or request approval. |
| `INVALID_STATE` | 5 | 409 | The transition guard refused (state not in `from`, or a terminal row); a declared handler's `assert` with `else.code: "INVALID_STATE"` failed; `on_delete: restrict` blocked a delete (`{ link, target, child_table, child_count }`, empty for a fenced caller); `declaration_changed`; the action's `offeredWhen` does not hold on the subject as you read it (`details.code: not_offered`). | Move the row through the allowed transitions first. |
| `CONFLICT` | 6 | 409 | Concurrency and lifecycle guards: two writers raced one idempotency key, an optimistic-concurrency check failed, an apply lease is held, a compound part write hit a frozen root, a generated write hit a record in a terminal state. | Switch on `details.code` / `details.reason` (below). A same-key retry returns the cached envelope, never `CONFLICT`. |
| `PAYLOAD_TOO_LARGE` | 2 | 413 | A file over the `max_bytes` its `file` property, input or type's `attachments` declare (`details.rule: "max_bytes"`, `max_bytes`, `size_bytes`), or an upload over the platform cap (`details.rule: "platform_max_bytes"`). | Upload a smaller file; `canon schema describe` shows a property's `max_bytes`. |
| `RATE_LIMITED` | 7 | 429 | Reserved: a per-principal or per-action rate budget. Not raised today; the shape is stable. | Back off per `details.retry_after_ms`. |
| `INTERNAL_ERROR` | 1 | 500 | Any non-domain failure (uncaught exception, infra). Not an `ActionDomainError` kind; never raise it. | A platform bug or outage — `canon report`. |

Two more HTTP-level codes exist: `UNAUTHENTICATED` (401) and
`SERVICE_UNAVAILABLE` (503).

**`INVALID_STATE` from a guard** carries `{ current_state, allowed_from,
object_type, id }` to a caller who may read the state column, and only
`{ object_type, id }` (with a generic message) to anyone else — a hidden
column's value never rides out on a refusal. **From a handler `assert`** it
carries the assert's own `message` and, when `else.field` is set,
`failed_precondition: { field }` — no `current_state` / `allowed_from`, since
the handler, not the guard, judged the state.

### Two `NOT_FOUND` shapes

- **Missing subject** — the row the action operates on. The transition guard
  reports `details: { object_type, id }`; a handler `read` / `update` /
  `delete` step reports `{ object_type, step }` (its `where` stays in the audit
  row). The runtime never pre-checks the subject; that is the handler's call.
- **Missing foreign reference** — a declared action's input naming another type's
  row (a declared belongs-to link, a `subject` of another type, an `inputFilters`-
  scoped input, an array row's `<x>_id`) that is absent, unreadable or outside its
  scope — all alike, before the handler runs: `message` says it in words
  (`lines[1].product_id does not name a product that can be chosen here. …`),
  `details: { field, row?, object_type, code: 'foreign_key_not_found',
  message_key, message_params }`; `field` (`lines.product_id`) to fix. A
  generated create / update / replace answers as the FOREIGN KEY row below.

### Constraint violations (`VALIDATION`)

| Situation | Result |
|---|---|
| A handler `insert` / `update` step names a parent id that does not exist (FOREIGN KEY), or a generated create / update / replace names one that a link declared on either side or a `REFERENCES` names, missing or unreadable (checked before any write) | `VALIDATION`, `details.code: constraint_violation`, with `details.field` (the property), `details.references` (the parent ObjectType) and `details.value` (the missing id); `field_errors[<field>]` = `violates a FOREIGN KEY constraint: no <parent> has id "<value>"`. That full refusal goes only to a caller whose read of the parent type is an unfiltered Allow; any other caller gets the field name only (`<field>` names a record that does not exist or that this caller may not read), with no `details.references` / `details.value`. |
| A UNIQUE, CHECK or NOT NULL violation | `VALIDATION`, a `message` naming the kind of constraint in words, `details.code: constraint_violation`, `message_key: '@t:errors.constraint_violation'`, `message_params: { constraint: 'unique' \| 'not_null' \| 'check' \| 'primary_key' }` — not `CONFLICT`. Find the existing row with `canon objects list <type> --where '{"<col>":"<value>"}'`. |
| An `objects.import` row | The batch is refused naming the row: `details.row`, `field_errors["rows.<i>.<field>"]`; a missing or unreadable parent follows the FOREIGN KEY row above. |
| A reference to an unknown step binding or property | `VALIDATION` at declare-time type-check. |
| `delete` against an `on_delete: restrict` link with children | `INVALID_STATE`. |

### `sql_read`

`READONLY` (not one read-only `SELECT`; exit 2, HTTP 400), `SQL_READ_ERROR`
(does not compile or run; exit 2, HTTP 400), `SCHEMA_CHANGED` (the schema moved
under the statement — run it again; exit 6, HTTP 409).

### Filter size limits (`VALIDATION`)

| Limit | `details.code` |
|---|---|
| deeper than 32 JSON levels (about 15 nested `$or` / `$and`), or more than 2 nested `$exists` / `$not_exists` | `where_too_deep` |
| more than 1000 members in one array, or 10000 values bound by the whole filter | `where_too_large` |

Aggregate reads cap at 4000 buckets: a `window.last`, `compare.offset.periods`
or `compare.rolling.buckets` above 4000 is refused at `view_specs.declare` and
at read time. Split a huge `in` list, or filter on a relation instead.

## `CONFLICT` sub-codes

| Discriminator | Where | Meaning and fix |
|---|---|---|
| `BASELINE_REQUIRED` | `catalog.apply` | No `baseline` and no `force`, on a canon that already holds declarations (a first apply to an empty org needs neither). Export, diff against your bundle, re-apply with the fresh `baseline` (from `catalog.export` / `manifest.baseline`; `canon catalog export <dir> --baseline-only` prints just the token) or `force: true`. |
| `STALE_CANON` | `catalog.apply` | Artifacts this apply would change or prune drifted since your baseline: `details.drifted: [{ key, current, baseline }]` (`key` like `action:<name>`); a re-provisioned database gives `reason: 'epoch_mismatch'`. Disjoint edits pass. Re-export, re-plan, re-apply. |
| `HASH_REQUIRED` | re-declaring an existing artifact: `actions.update`, `catalog.refine_object_type`, `view_specs.declare`, `state_machines.register`, `email_templates.declare`, `apps.declare` | No `expectedHash` and no `force`; `details.current_hash`. A first declare is never guarded. Read the `declaration_hash` from `catalog.export` (also `canon actions describe` for actions) and re-submit. |
| `STALE_DECLARATION` | same verbs | `expectedHash` is stale: `{ current_hash, expected_hash, drift }`, `drift` naming the changed top-level keys. Re-read, re-apply your intent, re-submit. |
| `APPLY_IN_PROGRESS` | `catalog.apply` | Another apply holds the org lease; nothing written. Retry shortly. |
| `APPLY_LEASE_LOST` | `catalog.apply` | This apply outlived its lease; writes may be partial. Re-export and re-apply. |
| `reason: 'FROZEN'` | compound part write | The root is frozen by `parts_frozen_when`: `{ reason, object_type, column, frozen_values, current_value }`. |
| `reason: 'TERMINAL_STATE'` | generated `update` / `replace` / `delete` and temporal `set` / `end`; on an owned child (its parent's link is `part_of` or `on_delete: cascade`, without `settles: false`) also the generated `create` | The record — or, with `details.owner: { object_type, id, link }`, the parent that owns it — is in a `terminal` state of a StateMachine: `{ reason, object_type, id, state_machine, property, terminal_states }`; nothing written. Correct it with a differently named declared action (`<type>.correct`); a hand-declared `<type>.update` first needs `defaults: { update: false }` ([state-machines.md](../../canonify-objects-actions/resources/state-machines.md#a-terminal-state-settles-the-record)). |
| no `reason` | compound write | The root version moved since you read it (optimistic check); re-read and retry. |

Pinning and forcing together — `baseline` + `force`, or `expectedHash` +
`force` — is contradictory intent: `VALIDATION`,
`CONTRADICTORY_BASELINE_INTENT` / `CONTRADICTORY_DECLARATION_INTENT`.

## Effective-dated (`temporal`) sub-codes

| Code | Kind | Where | Meaning |
|---|---|---|---|
| `temporal_overlap` | `VALIDATION` | `<type>.set` | Intersects an existing period of the key; `conflicting_id` / `_from` / `_to` name it (omitted when you may not read it). |
| `temporal_empty_period` | `VALIDATION` | `<type>.set` | `to` is not after `from`. |
| `temporal_end_before_start` | `VALIDATION` | `<type>.end` | `to` is not after the open period's `from`. |
| `temporal_not_open` | `VALIDATION` | `<type>.end` | The `id` names a closed period. |
| `temporal_key_required` | `VALIDATION` | `<type>.end` | No `id` and an incomplete key; `details.missing` lists it. |
| `temporal_no_open_period` | `NOT_FOUND` | `<type>.end` | Nothing open for the key. |
| `temporal_invalid` | `VALIDATION` | declare / apply | A bound or key member is not a mapped `string` / `timestamp`, repeats, or is `id`. |
| `temporal_open_rows_conflict` | `VALIDATION` | declare / apply | The table already holds two open periods for one key; close one first. |
| `as_of_not_temporal` | `VALIDATION` | `as_of` reads, `$as_of`, handler type-check | The type declares no `temporal`. |
| `as_of_invalid` | `VALIDATION` | `as_of` reads, `$as_of` | Not a date, ISO date-time, `$now` or relative-date token. |

## Bundle diagnostics — `canon catalog validate` and `catalog.apply`

`catalog.apply` runs the full validate pass before its first write. A refusal
there is `VALIDATION` with `details.apply_state: "pre_write"`: nothing was
written, the message leads with the first violation and
`details.diagnostics` lists every blocking one. A diagnostic is
`{ level, code, artifact, message }`:

- **`error`** blocks apply, always.
- **`warning`** (`unresolvable_title`, `missing_view_icon`, `missing_part_of`)
  blocks only under `strict` (`canon catalog validate --strict`, `canon catalog
  apply --strict`, `catalog.validate { strict: true }`); a normal validate exits
  0 with only warnings. `removed_property_not_dropped` never blocks, not even
  under `strict`. Nothing is raised about the platform's own ObjectTypes.

| Code | Means |
|---|---|
| `unresolvable_title` | (warning) A type's `display.title` would read empty or the raw id; the message names the part and why: a `{ property }` that is not a field, a template `{token}` that names no field (a link's name, `{customer}`, holds no value — write its FK, `{customer_id}`, to read the linked title), a `derived.via` that is not a field, or no title and no text field to infer one. A type refused for another error is reported for that error only. |
| `registry_content_mismatch` | An action handler addresses a property whose live type differs from the type the bundle's DDL derives (names action, `object_type`, `property`, `live_type`, `bundle_type`; the type error it causes follows). Converge the column with a migration or restate the property in the bundle's refinement. `force` does not override it. |
| `invalid_currency` | A `currency` that is not an uppercase ISO-4217 code (`kr` fails), a `money` property (derived ones too) without one, or a currency on a non-`money`/`number` property. A `money` action input without `currency` fails `actions.declare` instead (`invalid_artifact` in a bundle). |
| `derived_expression_invalid` | A derived property's `expression` does not compile in the type's read view (a missing column, or a sibling derived property named bare inside a subquery — qualify it with the table); names the property, the expression and the SQLite message. Also raised by `catalog.refine_object_type`. |
| `derived_cycle`, `derived_own_table_rebound` | Derived properties reference each other in a cycle (names the path, `a → b → a`); or a subquery re-reads the own table unaliased around a `table.derived` reference — alias it (`FROM job AS j2`). Also raised by `catalog.refine_object_type`. |
| `derived_sql_clock`, `derived_parameter_unknown` | A derived expression reads SQLite's clock (`date('now')`, `datetime('now')`, `julianday('now')`, `strftime(…, 'now')`, `unixepoch('now')`, `date()`, `CURRENT_DATE`, `CURRENT_TIMESTAMP`), which is UTC real time — write `:today` (the org's date) or `:now` (its now); the message shows the expression rewritten. Or it names a parameter other than `:today` / `:now`. Also raised by `catalog.refine_object_type`. |
| `invalid_condition` | A named condition (a derived boolean) is `on` or counts an `amount` that is not a field of its type of the right type (dates, timestamps, numbers; numbers or money), has an `amountLabel` without an `amount` or a `fixSubject` without a `fix`, or is fixed by an action that is not registered or acts on neither the type, its subject, nor the record `fixSubject` reaches. The message lists the fields or links to use. Also raised by `catalog.refine_object_type`. |
| `invalid_enum_color`, `invalid_enum_label`, `invalid_enum_values` | A `valueColors` / `valueLabels` key that is not one of `values`, a non-token color, a blank label; a `set<enum>` with no or duplicate `values`. |
| `invalid_enum_meaning` | A `valueMeanings` key not in `values`, a meaning outside `live` / `terminal-good` / `terminal-bad` / `sink` (a status) and `critical` / `warn` / `good` (a band), a mix of the two, or meanings on an enum a state machine governs (declare them on its states). |
| `invalid_property_label` | A blank `label` (omit it to get the humanized name). |
| `unknown_icon`, `invalid_action_override` | An ObjectType `icon` that is not a known Lucide icon; a `display.actions` override naming an unregistered action, an action of another ObjectType, or an invalid field. |
| `part_of_shape_invalid`, `part_of_forest_violated`, `settles_shape_invalid`, `rollup_invalid`, `parts_frozen_invalid`, `reserved_table_name` | Compound and schema shape rules ([object-links-and-cardinality.md](../../canonify-objects-actions/resources/object-links-and-cardinality.md)); a table named like a platform ObjectType. |
| `invalid_access_policy` | A policy that does not decode, names an ObjectType that does not exist, or whose `row_filter` compiles neither for a member nor an external fixture. An empty-string `row_filter` is refused (omit it). A wildcard resource with a filter is checked at read time. Columns the same bundle adds count as known. |
| `invalid_composite_filter` | A composite `filters` entry no direct child can honor: the property is on no child source, or the wrong type (`date-range` needs a `timestamp`, `dimension` an `enum`, `date` a `string` / `timestamp`, `reference` the `via` FK of a to-one link). |
| `duplicate_block_declaration` | One composite declares the same block twice (two children of one kind under one `title`, or identical apart from `title` / `label`); names `block` and `duplicate_of`. Siblings only. |
| `duplicate_block_heading` | Two blocks would paint the same heading (an untitled table paints its plural, a sibling metric is titled the same; or a block repeats a titled composite above it), in any locale the canon ships; two `detail`s of one type with the same `header.title` on one page (`relation: "record"`). Names `block`, `duplicate_of`, `heading`, `relation`, `locale`. Blocks in different tabs, differently titled sections or exclusive `showWhen`s are fine. Give one a distinct title. |
| `invalid_aggregate_window` | `window` without an aggregate source or a time-bucketed `groupBy`. |
| `invalid_aggregate_compare` | `compare` without that spine, beside `seriesBy` or a chart's measure list, over `avg` / `max`, or with a `source.filter` on the spine property (scope with `window`). |
| `invalid_aggregate_compare_offset` | `compare.offset.period` not admitted by the spine's bucket. |
| `removed_property_not_dropped` (warning) | A property in the stored refinement is absent from the new one; the column persists and is re-inferred. Drop the column (`ALTER TABLE … DROP COLUMN`) or re-declare the property. |

`view_specs.declare` refuses with these in `details.code` (and they are
error-level diagnostics in validate / apply): `invalid_filter_value_type` (a
filter value not matching the property type; booleans take `true` / `false` or
`0` / `1`, `money` whole minor units, enums a declared value; `set<enum>` cannot
be filtered this way), `invalid_lookup` / `invalid_measure_factor`,
`invalid_source_action` (a form `sources` binding a non-`read: true` action;
unknown is `unknown_action`), `unknown_detail_link` / `detail_link_not_many` /
`duplicate_detail_link` / `invalid_related_path` / `duplicate_primary_detail_link` / `duplicate_primary_list` (two `primary: true` blocks), `invalid_detail_totals`, `invalid_aggregate_table_column`, `invalid_queue_row`
(more than three `facts`, or a `row.peek` of another type or block; a table `peek`: `invalid_table_peek`), `detail_source_not_record_in_scope` /
`file_source_not_record_in_scope` (a composite child `detail`, or a `file`
block's `objectType`, of another type than the record the view shows).
`actions.declare` refuses `invalid_presentation` (a presentation key naming an
undeclared input, a non-token enum default, an `inputFilters` key that is
not a property of the picked type, a `$subject.<column>` the subject type lacks
or on an action with no subject, or `contains` on a property that is not a
`set<enum>` / `json`) and `link_path_to_many` (an `inputFilters`
key through a to-many link); a stored action failing either is skipped at
boot. `schedules.declare` refuses `input_template_not_text` (a `{token}` aimed
at an enum, number or boolean input), and boot skips a stored schedule that
does.

`canon catalog audit <dir>` adds the coherence rules: `C11-wildcard-grant`
blocks apply; `C1-shadow-entity` is a warning validate and apply report but
never block on (a name match); `C10-float-money` is an error-level authoring
diagnostic outside the blocking set. What each means and how to fix
it: the
[`canonify-domain-modelling`](../../canonify-domain-modelling/SKILL.md) and
[`canonify-app-lifecycle`](../../canonify-app-lifecycle/SKILL.md) skills.

## The CLI version floor — HTTP 426 `CLI_UPGRADE_REQUIRED`

Not an envelope: a request to `/v1/*` or `/api/auth/*` from a `canon` binary
older than the server's floor gets a flat body.

```json
{
  "error": "canon 2026.6.1.1200 is older than the minimum supported 2026.7.1.900 — run: canon upgrade",
  "code": "CLI_UPGRADE_REQUIRED"
}
```

Run `canon upgrade` (it downloads the current binary from `/cli/*`, which is
never gated, and swaps it in with a backup). The floor only moves forward: the
newest release older than 30 days, or a compatibility break forced sooner — any
build keeps working for at least 30 days. Before a server-bound command the CLI
checks `GET /cli/latest` (cached 24 h): below the floor it refuses locally with
the same message (offering `Upgrade now? [Y/n]` on a terminal); merely outdated
it prints a nag and proceeds. Dev builds are never gated.

## Audit status

| `status` | Cause |
|---|---|
| `success` | The handler returned. |
| `denied` | Governance (`FORBIDDEN`) or the transition guard (`INVALID_STATE`) refused; the action did not run. |
| `error` | Every other failure. |
| `sensitive` | A granted mutation on a `confidential` type. |

Shell automation can switch on the exit code: `canon …; case $? in 3) …`.
