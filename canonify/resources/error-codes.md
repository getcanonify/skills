# Error codes — `ActionDomainError` kinds

Every Canonify error envelope carries a stable `error.code` string the
caller can switch on. The runtime defines them in
[`packages/canon/src/runtime.ts`](../../../../packages/canon/src/runtime.ts).
Handlers raise them by throwing
`ActionDomainError(code, message, details?)`; the runtime catches the
class, rolls the tx back, writes an audit row with the appropriate
status, and renders the error into the envelope's `error` field —
preferable to translating every domain failure into `INTERNAL_ERROR`.

Envelope shape when a domain error fires:

```json
{
  "data": null,
  "audit_event_id": "aud_…",
  "approval_status": "not_required",
  "error": {
    "code": "NOT_FOUND",
    "message": "lead not found (step 0: lead)",
    "issues": [],
    "details": { "code": "lookup_failed" }
  }
}
```

The CLI maps each kind to a deterministic exit code; see
[Phase 17.6 §7](../../../phases/17.6-cli-ergonomics.md#7-output-modes--error-contract).

### Structured field slots (additive)

Two OPTIONAL first-class slots ride next to `code`/`message`/`issues`/`details`
when the failure attaches to a named input field. Both are **additive** — an
error that carries neither is byte-identical to the older shape — and the
`code`/`message` strings are unchanged (they remain the stable, English wire
contract; display localization is a client concern):

- **`field_errors`** — `{ "<field>": "<message>" }`. Populated on every
  pre-handler input-validation failure (derived from the same `issues` array,
  so callers no longer need to parse issue paths), and on any handler-raised
  error whose `details.field_errors` carries a map (the runtime lifts it to
  this slot). Render these inline on the offending field.
- **`failed_precondition`** — `{ "field": "<input param>" }`. A precondition
  failure's reference to the input field whose check failed. Raised by a
  handler-DSL `assert` step that declares `else.field` (optional; see the
  handler-DSL cookbook) — the author's verbatim `message` is the copy, the
  `field` names where to attach it.

Machine params for the other high-traffic kinds ride in `details`:
`INVALID_STATE` carries `{ current_state, allowed_from, object_type, id }`
(the last two are additive), the FSM missing-row `NOT_FOUND` carries
`{ object_type, id }`, and a principal-kind governance denial carries
`details.reason: 'PRINCIPAL_KIND_NOT_ALLOWED'` with
`{ principal_kind, action, allowed }` (see `FORBIDDEN` below). Clients
SHOULD key display copy off `code` + these params and fall back to the
English `message` for anything they don't recognise.

---

## The kinds

| Code | CLI exit | When it fires | Agent recovery hint |
|---|---|---|---|
| `VALIDATION` | 2 | Schema decode of input failed; handler DSL type-check failed at `actions.declare`; handler writes the FSM-bound state column when a `transition` marker is set (`direct_state_write`); auto-CRUD detects a malformed declared property. | Call `canon actions describe <name> --json` (or `--explain`) to fetch the input JSON Schema; correct the offending field and retry. For `actions.declare` typecheck failures the `details.errors` array names the offending step and field. |
| `NOT_FOUND` | 3 | `read` step found zero rows for its `where` clause; `runtime.get(type, id)` was called with an unknown id; `runtime.invoke(name, …)` was called with an unregistered action name; `catalog.refine_object_type` targeted an ObjectType that isn't registered. | Verify the id exists via `canon objects get <type> <id>` or list the candidates via `canon objects list <type>`. For unregistered actions, `canon actions list` is the discovery surface. |
| `FORBIDDEN` | 4 | Governance gate denied the call — the caller's principal kind isn't in `governance.allowed_principals`, OR an approval-required action was invoked without an approved record. | Check the action's governance via `canon actions describe <name> --json`. If the action requires approval, request one (Phase 22 surfaces). If the principal kind is wrong, switch identities (`canon auth login` as a different role, or use a service-account key). |
| `INVALID_STATE` | 5 | Phase 18.5 FSM transition guard fired: the row's current state is not in the action's `transition.from` set; the row is in a terminal state and any non-terminal-targeting transition was attempted; `on_delete: restrict` blocked a delete because child rows exist. | The envelope's `details` carries `{ current_state, allowed_from }` for FSM denials. For cascade restricts, the details include the child link name and count. Either move the row through the allowed precursor transitions first, or pick a different terminal path. |
| `CONFLICT` | 6 | A unique-constraint violation hit the storage layer (duplicate id/email/etc.); two writers raced for the same idempotency key and one lost; a `catalog.apply` optimistic-concurrency / lease check failed; a compound part write hit a frozen root. Covers storage-layer concurrency and the compound-write guards (ADR 0002 §5); the runtime translates SQLite `SQLITE_CONSTRAINT_UNIQUE` failures into this kind. | Switch on `error.details.code` (or `.reason`) for the specific cause — see the sub-codes table below. For a plain unique collision: retry with a fresh idempotency key, OR adjust the input so the unique field no longer collides (`canon objects list <type> --where '{"<col>":"<value>"}'` to find the existing row). Idempotent re-submits with the **same** key return the cached envelope, not a `CONFLICT`. |
| `RATE_LIMITED` | 7 | The caller has exceeded a per-principal or per-action rate budget. Reserved for the governance hardening phase; not raised by slice-1 today but the envelope shape is stable so callers can switch on it. | Back off per the `details.retry_after_ms` field (when present) and retry. If you're an automation, lower the dispatch concurrency before retrying. |

### `CONFLICT` sub-codes

`CONFLICT` is the catchable top-level `error.code`; the specific cause
rides in `error.details`. These are **not** separate top-level codes —
a caller switches on `error.code === 'CONFLICT'` first, then inspects
the discriminator (`details.code`, or `details.reason` for the freeze
gate). All raised by the compound-write / `catalog.apply` machinery
(ADR 0002 §5, ADR 0009).

| Discriminator | Where | Meaning & recovery |
|---|---|---|
| `details.code: 'BASELINE_REQUIRED'` | `catalog.apply` | You supplied **no** `baseline` and no `force`. The check is default-on, so apply refuses to write blind rather than last-writer-wins. Export the canon and diff it against your bundle — you may be reverting work applied after you exported — then re-apply with the `baseline` from that fresh export (surfaced on `catalog.export` stdout / `manifest.baseline`), or pass `force: true` to overwrite deliberately. |
| `details.code: 'STALE_CANON'` | `catalog.apply` | One or more artifacts your apply would **change or prune** have drifted since your `baseline` (someone edited them in between). `details` carries `{ drifted: [{ key, current, baseline }] }` (per-artifact: `key` is `action:<name>` / `view:<name>` / …); a re-provisioned DB instead carries `{ reason: 'epoch_mismatch' }`. Disjoint edits pass — only artifacts this apply touches conflict. Re-export the canon, re-diff, re-plan against the drifted artifacts, re-apply with the fresh baseline. |
| `details.code: 'HASH_REQUIRED'` | `actions.update`, `catalog.refine_object_type` (re-refine of an already-refined type), `view_specs.declare` (re-declare of an existing name), `state_machines.register` (re-register of an existing SM), `email_templates.declare` (re-declare of an existing `(template_id, locale)` pair), `apps.declare` (re-declare of an existing slug) | You supplied no `expectedHash` and no `force` on a re-declare of an artifact that already exists. Same default-on contract as `catalog.apply`'s `baseline`, one artifact at a time — a FIRST declare of a fresh name/slug/pair has nothing to conflict with, so the guard never fires there. `details` carries `{ current_hash }`. Read the artifact's current `declaration_hash` off `catalog.export`'s artifact record — the reliable path for all six verbs (`actions.describe` additionally surfaces it for `actions.update`). Confirm you are not reverting a newer declaration, then re-submit with that `expectedHash` — or `force: true`. |
| `details.code: 'STALE_DECLARATION'` | Same verb set as `HASH_REQUIRED` above | The `expectedHash` you passed doesn't match the artifact's current persisted declaration — it changed under you. `details` carries `{ current_hash, expected_hash, drift }`, where `drift` names the top-level declaration keys that differ (e.g. `['input','handler']` for an action, `['spec']` for a View). Re-read the declaration, re-apply your intent on top of it, re-submit with the fresh hash. |
| `details.code: 'APPLY_IN_PROGRESS'` | `catalog.apply` | Another apply is holding the org-scoped apply lease. No write happened. Retry shortly. |
| `details.code: 'APPLY_LEASE_LOST'` | `catalog.apply` | This apply ran longer than the lease TTL and a competing apply took over; its writes may be partial. Re-export and re-apply. |
| `details.reason: 'FROZEN'` | compound part write | A part insert/update/delete was attempted on a compound root that is frozen by its `parts_frozen_when` gate. `details` carries `{ reason, object_type, column, frozen_values, current_value }`. Move the root out of the frozen state (or don't mutate parts of a finalised root). |

**Pinning and forcing are mutually exclusive.** Supplying `baseline`
*and* `force: true` to `catalog.apply` (or `expectedHash` and `force: true` to
any of the six verbs above) is contradictory intent — you are both pinning a
version and demanding the pin be ignored — so it is rejected as a `VALIDATION`
error (`details.code: 'CONTRADICTORY_BASELINE_INTENT'` /
`'CONTRADICTORY_DECLARATION_INTENT'`), **not** a `CONFLICT`. Pick one. Note the
status asymmetry when writing a client: `VALIDATION` surfaces over REST as HTTP
`400`, whereas the `CONFLICT` sub-codes above ride a **`200`** envelope — switch
on `error.code`, never on the transport status.

`INTERNAL_ERROR` (CLI exit `1`) is the catch-all for any non-domain
failure — uncaught exception, infra failure, DB connection drop. It is
**not** an `ActionDomainError` kind; handlers should never raise it
explicitly. If you see one in production, it's a bug in the runtime or
a real outage, not an agent input problem.

---

## Canon-validate diagnostics — `error` vs `warning`, and `strict`

`canon catalog validate` (and the same gate inside `catalog.apply`) emit
**diagnostics**, a *separate* concept from the envelope `error.code` kinds
above. A diagnostic is a `{ level, code, artifact, message }` finding about a
**bundle's quality**, not a runtime failure. `level` is the load-bearing field:

- **`error`** — a real defect that would render broken or apply an incoherent
  shape. Error-level diagnostics **block apply** unconditionally — even a
  non-strict apply. Examples: `unknown_icon`, `invalid_enum_color`,
  `invalid_enum_label`, `invalid_property_label`, `invalid_action_override`,
  `invalid_currency`, `invalid_composite_filter`, `invalid_metric_target`,
  `part_of_shape_invalid`,
  `part_of_forest_violated`, `rollup_invalid`, `parts_frozen_invalid`,
  `invalid_access_policy`.
- **`warning`** — a report-only nudge. Warnings **do not block** a normal apply;
  they block **only under `strict`**, where **any** diagnostic (warnings
  included) fails the gate. Examples: `unresolvable_title`, `missing_view_icon`,
  `missing_part_of`, and the advisory `elegance_*` lints. One warning is exempt
  from the `strict` promotion — see `removed_property_not_dropped` below.

So the apply gate's rule is: **block on any `error`; under `strict`, block on any
diagnostic at all — except `removed_property_not_dropped`, which never blocks.**

### `invalid_currency` (error)

A `number` property's declared `currency` isn't a three-letter ISO-4217 code (the
shape check is `/^[A-Z]{3}$/` — a symbol like `kr` or a suffix fails), **or** a
`currency` is declared on a **non-`number`** property. Use an uppercase code
(`DKK`, `EUR`, `USD`) on a `number` property. See the base skill's *Property
currency*.

### `invalid_enum_label` (error)

An enum property's `valueLabels` map either names a **key that isn't one of the
property's `values`** (a label for a value the column can never hold), or maps a
value to a **blank / whitespace-only label** (which would render an empty cell).
The sibling of `invalid_enum_color`: labels are otherwise free text (no closed
palette), so the only checks are key-membership and non-emptiness. Fix the key or
supply a non-empty label. See the base skill's *Enum value labels*.

### `invalid_composite_filter` (error)

A `composite` ViewSpec's `filters` axis (the reader-adjustable filter row)
declares a filter that no child can honor. It fires when the named `property`
**exists on no direct child source**, or when the property is the wrong type for
the declared `kind` — `date-range` requires a **`timestamp`** property on some
child, `dimension` requires an **`enum`**. The message names the view and the
offending property. Point the filter at a real child-source property of the
matching type (a `timestamp` for `date-range`, an `enum` for `dimension`), or
remove the filter. See the viewspecs skill's *Reader-adjustable filter bar*.

### `invalid_metric_target` (error)

A `metric` ViewSpec declares a `target` (the gauge/bar fill max) on a `display`
that renders no meter. `target` drives the value/target fill fraction, and only
`display: "gauge"` or `"bar"` draws that meter — on `"scalar"` (the default) and
`"hero"` a declared `target` decodes and stores but draws **nothing**. It fires
whenever `target` is present and `display` is not `"gauge"` or `"bar"`; the
message names the view and the offending display. Set `display: "gauge"` or
`"bar"` to render the meter, or drop `target`. See the viewspecs skill's *Gauge /
bar display*.

### `invalid_access_policy` (error)

A bundle-declared access policy (the `_access_policy` records a `policies`
section applies) fails one of the checks `catalog.validate`/`catalog.apply`
now run at declare time (tick s2l): the definition doesn't decode (unknown
operation, wrong-shaped `columns`), its `resource.object_type` doesn't exist
in the catalog, or its `row_filter` doesn't compile. Before this, a malformed
`row_filter` shipped green and only failed at READ time in production
(fail-closed, but an outage discovered live) — this closes that gap by
running the same compiler at declare.

Rules worth knowing when fixing one:

- A filter is rejected only if it fails to compile for **both** a member
  fixture **and** an external fixture — `:actor_id` / `:actor.<attr>` stay
  legal (an external principal can match a `{role}` subject), so per-kind
  bindability is still the runtime's own loud reject, not a declare-time one.
- A **wildcard resource** (no `object_type`) with a `row_filter` is legal and
  skipped at declare — there's no single type to compile against; the
  runtime still gates it per-type at read.
- An **empty-string** `row_filter` is rejected outright — omit the field
  entirely for an unfiltered grant rather than declaring `""`.

The message names the policy and the specific grammar/column/token problem;
fix the filter and re-validate/re-apply. Via `catalog.apply` this surfaces as
a top-level `VALIDATION` failure (checked before any write commits); via
`canon catalog validate` it's one entry in the `diagnostics` array with this
code.

### `removed_property_not_dropped` (warning, never blocks — not even under `strict`)

`catalog.apply` fires this when a property present in the **stored**
`catalog.refine_object_type` declaration is **absent** from the incoming one.
Removing a property from a declaration can't drop data: the physical column
persists and auto-CRUD re-infers the property from it on every boot, so a live
org silently diverges from a fresh org built off the same bundle (the property
re-appears there too). The message names the object type, the property, and
the persisting column, and points at the convergence path — drop the column
with `schema.propose_migration` (`ALTER TABLE ... DROP COLUMN`), or re-declare
the property to keep it. Unlike every other warning on this page, `strict`
does **not** promote this one to a blocker: drift must stay observable without
ever wedging a CI apply. See the app-authoring skill's *apply is fully
declarative — it prunes*.

### `invalid_property_label` (error)

A property declares a `label` that is **blank or whitespace-only**, which would
render a nameless field — strictly worse than the humanised fallback it
suppresses. Non-emptiness is the **only** rule: a label is otherwise opaque
authored text, so there is deliberately no charset, casing, or length check.
Supply a real label, or omit the field entirely to fall back to humanisation of
the property name. See the app-authoring skill's *Property display label*.

### `elegance_*` (warning, advisory)

The **elegance authoring lint** — a high-signal set of `warning`-level hints that
nudge you toward presentation knobs the platform offers but this canon leaves
unused. They are **purely advisory**: they never block a normal apply, and each
message carries the fix.

| Code | Fires when | Fix it suggests |
|---|---|---|
| `elegance_status_no_colors` | a status / enum property (the top one per object) has no `valueColors` | map its values to palette tokens so status chips render semantic colors |
| `elegance_detail_no_header` | a `detail` view over a status-bearing object declares no `header` | add `header: { title, subtitle, status }` field refs |
| `elegance_list_no_presets` | a `table` over a status-bearing object declares no filter `presets` | add `presets: [{ label, filter }]` (e.g. one chip per status) |
| `elegance_numeric_no_metric` | a numeric / rollup object has a detail view but no `metric` tile surfacing its figures | add a `metric` view as a headline stat |
| `elegance_unlinked_child_table` | a record-scoped `composite` includes a child `table` with no filter, and the parent has no many-link to the child type — so the table stays org-wide inside the record page | declare a `cardinality: 'many'` link parent→child (auto-scopes via its FK), or add an explicit `source.filter` scoping the table to the record |
| `elegance_missing_value_labels` | an enum property declares `valueColors` but no `valueLabels`, so its chip renders the raw token (`green`) not a human label | add `valueLabels` mapping each value to display text (the same site as `valueColors`) |
| `elegance_ungrouped_action_wall` | an ObjectType has more non-CRUD actions than the toolbar's inline budget and none declares `presentation.group`/`emphasis` — a flat button wall | group related actions under a shared `presentation.group` and mark the primary one `emphasis: 'primary'` |
| `elegance_no_detail_view` | a rich record (≥3 many-links) has no declared `detail`/`composite` view anywhere, so row clicks land on the auto page | declare a `detail` view (or a `composite` containing one) for the type |
| `elegance_numeric_boolean` | a `number`-typed property whose name reads as a boolean flag (likely a 0/1 INTEGER) | declare `type: 'boolean'` instead — literal 0/1 writes still apply and reads render Yes/No |

---

## The CLI version floor — HTTP 426 `CLI_UPGRADE_REQUIRED`

This one is **not** an `ActionDomainError` kind and does not carry the
envelope shape above — it fires before routing, on any request to
`/v1/*` or `/api/auth/*` whose `canon` binary is too old to trust. You'll
see it as an HTTP `426 Upgrade Required` with a flat body (no `data` /
`audit_event_id` / `approval_status`):

```json
{
  "error": "canon 2026.6.1.1200 is older than the minimum supported 2026.7.1.900 — run: canon upgrade",
  "code": "CLI_UPGRADE_REQUIRED"
}
```

**What it means.** Every `canon` request carries its build version in the
`X-Canon-Version` header. The server compares it against a floor
(`MIN_CLI_VERSION`) stamped at deploy time; a binary older than the floor is
rejected outright rather than being let through to fail confusingly later
(an old build hitting the CSRF origin guard used to print `[object
Object]` and fully block the user — this floor exists so the failure is
legible instead). `/cli/*` (the download/update endpoints) is never
gated — that's the escape hatch a below-floor binary needs to reach in
order to fix itself. A dev build (`0.0.0-dev`) is never floor-gated
either.

**The remedy.** Run `canon upgrade` — it downloads the current binary
for your platform from `/cli/download/<asset>` and swaps it in place
(with a backup-and-restore on failure). That's the exact command both
this error and the CLI's own preflight nag print.

**The floor policy — a 30-day ratchet.** The floor is `max(acute floor,
time-ratchet floor)`, recomputed every deploy: the acute value is a
package-level compatibility break a PR can force (`packages/cli/compat.json`
→ `min_supported`); the time ratchet is the newest `cli-*` release tag
that is **more than 30 days old**. In practice: any given build keeps
working for at least 30 days after release, then becomes eligible to be
retired as the floor moves forward. The floor only ever moves forward —
nothing lowers it; when no acute value is set for a release, only the
time ratchet applies.

**The CLI's own preflight.** Before a server-bound command runs, `canon`
checks its own version against `GET /cli/latest` (cached for 24h in
`~/.canon/version-check.json`, so this isn't a network round-trip on
every invocation). Two outcomes below latest:

- **Below the floor** — the CLI refuses locally with the identical
  message the 426 above would produce, *before* attempting the network
  call that would 426. On an interactive terminal it offers `Upgrade
  now? [Y/n]` and runs the upgrade for you on `Y`.
- **Merely outdated (but still ≥ floor)** — a yellow nag to stderr
  (`Update available: X → Y. Run 'canon upgrade' to update.`) and the
  command proceeds normally.

Offline or a failed version-check fetch always proceeds — the server
still enforces the floor regardless, so nothing below it can silently
succeed.

---

## How handlers raise these

Handler authors don't usually throw directly — the declarative DSL
fires the appropriate kind from the executor:

| DSL situation | Raises |
|---|---|
| `read` step's `where` returns 0 rows | `NOT_FOUND` |
| `insert`/`update`/`delete` step hits a FK or unique constraint | `CONFLICT` (unique) or `VALIDATION` (other constraint) |
| Reference to a bound step that doesn't exist | `VALIDATION` (caught at declaration-time typecheck) |
| Reference to a property the source ObjectType doesn't have | `VALIDATION` (caught at declaration-time typecheck) |
| `delete` against an `on_delete: restrict` link with children | `INVALID_STATE` |

Custom platform-tier handlers (TS code, not the DSL) raise
`ActionDomainError` explicitly — see
[`packages/canon/src/actions/`](../../../../packages/canon/src/actions/)
for examples.

---

## Audit status mapping

Every invocation writes an `_audit_event` row regardless of outcome.
Status field:

| Audit `status` | Cause |
|---|---|
| `success` | Handler returned without throwing |
| `denied` | Governance gate refused (FORBIDDEN), or FSM guard refused (INVALID_STATE) |
| `error` | Any other failure — VALIDATION, NOT_FOUND, CONFLICT, RATE_LIMITED, INTERNAL_ERROR |

`denied` is reserved for guard rejections (action did not run); `error`
covers everything else that surfaces a non-success envelope. The
distinction matters for downstream analytics — denials are signals
about access control, errors are signals about input or infra.

---

## Reading the envelope programmatically

```ts
import type { Envelope } from '@canonify/canon';

const env: Envelope<MyData> = await runtime.invoke('lead.convert', { lead_id });

if (env.error !== undefined) {
  switch (env.error.code) {
    case 'NOT_FOUND':    return retryWithFreshLeadId();
    case 'INVALID_STATE':
      // env.error.details = { current_state, allowed_from }
      return walkLeadThroughFunnel(env.error.details);
    case 'FORBIDDEN':    return escalateForApproval();
    case 'VALIDATION':   return repairInput(env.error.issues);
    case 'CONFLICT':     return retryWithNewIdempotencyKey();
    case 'RATE_LIMITED': return backOff(env.error.details?.retry_after_ms);
    default:             throw new Error(`unexpected: ${env.error.code}`);
  }
}
// env.data is typed; env.audit_event_id is the durable record.
```

The CLI sugar collapses this into exit codes — `canon …; case $? in 3) …` —
which is the load-bearing shape for shell-script automation.

---

## See also

- [Phase 17.6 §7](../../../phases/17.6-cli-ergonomics.md#7-output-modes--error-contract) — CLI exit-code table
- [Phase 17.5 §4](../../../phases/17.5-declarative-handlers.md#4-the-runtime-executor--packagescanonsrchandler-dslexecutorts) — how the executor raises these kinds from DSL steps
- [Phase 18.5](../../../phases/18.5-state-machines.md) — FSM guard semantics and the `current_state` / `allowed_from` detail shape
- [Handler DSL cookbook](handler-dsl-cookbook.md) — which DSL constructs raise which kinds
