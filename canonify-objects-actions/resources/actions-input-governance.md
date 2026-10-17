# ActionTypes — `actions.declare`

`actions.declare` registers a custom ActionType from JSON. The handler (the
body) is [actions-handler-patterns.md](actions-handler-patterns.md); this page
is everything around it. A declared action is live on every surface at once, no
restart, and the CLI grows per-action sugar from its input
(`canon task.start --task_id tsk_…`).

## The declaration

```json
{
  "name": "task.start",
  "description": "Move an open task into in_progress and stamp the start time.",
  "object_type": "task",
  "input": { "task_id": { "type": "string", "prefix": "tsk_" } },
  "governance": {
    "requires_approval": false,
    "allowed_principals": ["user", "agent", "service_account"]
  },
  "idempotency": "none",
  "transition": { "state_machine": "Task.status", "from": ["open"], "to": "in_progress" },
  "handler": {
    "steps": [
      { "op": "update", "object": "task", "where": { "id": "$input.task_id" }, "set": { "started_at": "$now" } }
    ],
    "returns": { "task_id": "$input.task_id" }
  },
  "presentation": { "label": "Start", "emphasis": "primary", "success": "Task started." }
}
```

```sh
canon actions invoke actions.declare --input @task-start.json
```

Over REST the body is `{ "input": <the declaration> }` — the outer `input` is
the transport's, the inner one the declaration's. Success returns
`{ data: { name, registered: true, advisories? }, audit_event_id, … }`;
`advisories` lists authoring advice the declaration passed with (an
unresolved subject, below). The handler never writes `status`: the transition
marker does.

| Field | Required | Meaning |
|---|---|---|
| `name` | yes | `<object>.<verb>`, matching `^[a-z][a-z0-9_]*\.[a-z][a-z0-9_]*$`. The name `platform` and anything starting `platform.` is refused (`reserved_action_name`): that prefix marks control-plane events in the audit trail. |
| `description` | yes | One sentence for agents (`actions list`, `--explain`). Never shown in the app — that is `presentation.description`. |
| `object_type` | yes | One of **your** ObjectTypes (catalog grouping, surfaces). |
| `input` | yes | Field → descriptor (below). |
| `governance` | yes | Who may call it, approval (below). |
| `idempotency` | yes | `none` \| `per-key` \| `natural-key` (below). |
| `transition` | no | State-machine transition marker (below). |
| `subject` | no | `{ param, object_type }` — which input is the operated-on record (below). |
| `target` | no | External-write scope target (below). |
| `handler` | yes | `{ steps, returns?, on_fail? }`. |
| `fixture` | no | `{ principal, args, expectedEnvelope? }` — the conformance fixture; `args` must satisfy `input`, `expectedEnvelope` is partial. Recommended: without it the action cannot be checked across surfaces. |
| `presentation` | no | How a generated UI shows it (below). |

Any other field fails the decode. The platform-only ActionType flags —
`read` (skip the change-stream row), `reaction_safe` (a vetted reaction
target), `sensitive_fields` (redacted in the audit copy), `persistArgs` (slim
the stored args) — cannot be declared; needing one means the action belongs to
the platform.

**Read a platform type, never act on one.** An `object_type`,
`target.object_type` or `subject.object_type` naming a platform ObjectType
(`org_member`, the e-signature types, `_audit_event`, any `_`-prefixed table),
or a handler step that writes one, is refused (`VALIDATION`,
`details.code: platform_object_type_refused`). A read step over one is allowed
and decided when the action runs: only if that principal could read the type
itself unfiltered, else `FORBIDDEN` / `details.reason:
PLATFORM_READ_NOT_AUTHORIZED` — always refused when running unattended as the
system principal (a schedule target, a hook, a timer, an `on_fail`).

## `input`

A flat record of field → descriptor. **This table is the one list of input
types.**

<!-- input-types:begin -->
| `type` | Accepts | Modifiers |
|---|---|---|
| `string` | a string | `enum` (narrows to those literals), `prefix`, `min` / `max` (length), `format` |
| `number` | a finite number (a numeric string is coerced) | `min`, `max` |
| `money` | a safe **integer of minor units** (`125000` = 1,250.00); a fraction is refused, never rounded | `currency` (required), `min`, `max` |
| `boolean` | `true` / `false` (also `"true"`, `"false"`, `1`, `0`) | — |
| `date` | a calendar date `YYYY-MM-DD` | — |
| `json` | any JSON value | — |
| `set<enum>` | an array of distinct members of `enum` | `enum` (required) |
| `file` | a string referencing an uploaded file | — |
| `signature` | a string carrying a captured signature (e-signature flows) | — |
| `array` | a list of objects, each shaped by `items` — a map of field → descriptor whose types are `string` / `number` / `boolean` / `json` / `file` — or a list of enum strings when `items` is `{ "type": "string", "enum": [...] }` | `items` (required) |
<!-- input-types:end -->

Every field also takes `optional: true` (may be omitted) and
`nullable: true` (may be `null`). `format` applies to a `string` only and is
one of `email`, `iso8601` or `date`: `iso8601` is a UTC date-time such as
`2026-10-20T16:00:00.000Z`, `date` is `YYYY-MM-DD`. There is no `enum` input
type (an enum input is a `string` with `enum: [...]`) and no `timestamp` input
type (use a `string` with `format: "iso8601"`). A modifier on a type that does
not list it is refused at declare time.

```json
{
  "bike_id":     { "type": "string", "prefix": "bik_" },
  "priority":    { "type": "string", "enum": ["normal", "rush"] },
  "promised_at": { "type": "string", "format": "iso8601" },
  "quote":       { "type": "money", "currency": "DKK", "min": 0 },
  "notes":       { "type": "string", "optional": true }
}
```

A `money` input must declare an ISO-4217 `currency` (refused without one,
naming the input; refused on any other type). It fixes the minor-unit exponent
a generated form needs to turn the `1250` a person types into `125000`; the
wire value stays the integer. Use the currency of the property the handler
writes. `prefix` makes the runtime check an id starts with it. A bad input at
invoke time is `VALIDATION` with the issues in `error.issues`.

**A verb binds its own constant.** Model intent, not a setter: two zero-input
verbs `flag_at_risk` (`set: { at_risk: 1 }`) and `clear_at_risk` beat one
`set_at_risk` taking `at_risk` — the subject is then the only input, so the
button is one tap, while a required free-form field makes a form a person must
fill. Keep an input only for a genuinely free value (a price), with a default.

## `subject`

The subject is the input carrying the operated-on record's id. One rule
resolves it, first match wins: a declared `subject: { param, object_type }` >
an input named `<object_type>_id` (`task.start` → `task_id`) or a required
`id` > the single required input that references the action's own type,
whatever its name (`support_ticket.reply` → `ticket_id`) — the handler
addresses a row of that type by it (`where: { id: "$input.ticket_id" }`), it
fills a column holding a link to that type, or its `prefix` is that type's id
prefix. Two inputs referencing the own type are never guessed between. An
action whose id inputs all reference other types (a plain create, `job.book`)
has none; declare one to bind a create to its parent (`job.check_in`, subject
`customer`: [patterns.md](../../canonify-app-authoring/resources/patterns.md)).
When an action on T takes a required id input (`id`, `*_id`, or a `prefix`),
resolves no subject and inserts no T, no record would offer it:
`actions.declare` succeeds with an advisory naming the fix (`declare subject:
{ "param": …, "object_type": "T" }`) and `catalog validate` / `apply` warn
`unresolved_action_subject`. Discovery publishes the subject (`subject` on
each action, every surface) beside `creates` — the types a call inserts rows
of (the handler's `insert` steps; a generated create's own type) — and the web
binds it from context — a record page, a table row, a record-scoped view — so
a view never hand-wires
`presetParams: { task_id: "$context.id" }`. It is not a server-side injection:
CLI/REST/MCP callers still pass it.

## `governance`

| Field | Meaning |
|---|---|
| `requires_approval` | `true` parks every call: the envelope returns `approval_status: 'pending'` with `request_id` (`apr_…`, also `data.request_id`). |
| `requires_approval_when` | A where-DSL predicate (`and`/`or`/`cmp`/`in`/`is_null`) over the input's top-level keys and `$principal.<field>` (`kind`, `id`, `org_id`, `roles`, `groups`): `"amount_cents > 100000"`, `"$principal.kind = 'agent'"`. When true, that call is parked. |
| `allowed_principals` | Non-empty subset of `user`, `agent`, `service_account`; another kind gets `FORBIDDEN`. |
| `operation` | The policy operation the invoke gate checks (below). |
| `policy` | Optional named policy reference. |

A principal holding `manage` settles a parked call with `actions.approve {
request_id }` or `actions.reject { request_id, reason }`. Approval re-runs the
original call — as the requester for an ordinary action, as the approver for a
catalog declaration. The platform parks an **agent's** `actions.declare`,
`catalog.refine_object_type`, `view_specs.declare`, `org.delete` and
`org.restore` this way; nothing changes until approved.

The other principal kinds are server-minted, never listed here: `app`,
`external` and `public` reach your org's own actions (and the verbs over your
own types) through their access manifest or form policy — never platform
actions or verbs on platform types unless the platform opens them. `system` is
the cron identity that runs schedule targets, reaction actions and automatic
transitions; it cannot be claimed through any header, flag or env.

### `operation`

By default the gate derives the operation from the name: `<type>.create` →
`create`, `.update` → `update`, `.delete` → `delete`, anything else →
`invoke`. `governance.operation` overrides it with one of `get` · `list` ·
`navigate` · `aggregate` · `create` · `update` · `delete` · `invoke` ·
`manage`. `manage` is owner-equivalent (owner and admin resolve it, a member
does not): pin it when an action's effect is more privileged than its name —
`catalog.refine_object_type` itself declares `manage` so a member cannot bypass
the `catalog.apply` gate.

```json
{
  "name": "billing_config.rotate_key",
  "description": "Rotate the org billing key — owner-equivalent authority.",
  "object_type": "billing_config",
  "input": { "config_id": { "type": "string" } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent"], "operation": "manage" },
  "idempotency": "none",
  "handler": {
    "steps": [
      { "op": "update", "object": "billing_config", "where": { "id": "$input.config_id" }, "set": { "key_rotated_at": "$now" } }
    ]
  }
}
```

### Unattended work runs on its declarer's current authority

Schedules, reactions and a state machine's `automatic` transitions fire as
`system`, on the authority of whoever declared them — recorded from the
declaring call, re-checked before **every** fire. At declare time the declarer
must be allowed to invoke the target unfiltered and to `list`, unfiltered, the
scanned type and every type its `where` relates to (a reaction's `via` must
instead be `reaction_safe`; its `on` scan is checked at declare, related types
when it fires); none may scan or target a platform type. Before each fire,
if the declarer's key or membership lost that authority, nothing runs and
`schedules.history` records a run with one `FORBIDDEN` error whose message
starts with the reason:

| Reason | Meaning |
|---|---|
| `CRON_DECLARER_STANDING_LOST` | key revoked, member deactivated, re-roled or regrouped |
| `CRON_TARGET_NOT_INVOCABLE_BY_DECLARER` | may no longer invoke the target unfiltered |
| `CRON_SCAN_NOT_READABLE_BY_DECLARER` | may no longer `list` a scanned type unfiltered |
| `CRON_DECLARER_UNRECORDED` | no declarer recorded — fails closed |
| `CRON_DECLARER_MALFORMED` / `CRON_DECLARATION_MISSING` / `CRON_DECLARATION_INVALID` | stored declarer or declaration unreadable, gone, or its `where` no longer compiles |

A refused fire retries on the next cadence; an unreadable standing (outage)
records nothing. **Re-declare to resume** — that records you as the declarer.
Anyone with `manage` on `schedule` may re-declare a schedule; a reaction is
replaced or deleted only by its declarer or an owner/admin
(`REACTION_NOT_OWNED`). A machine registered by `catalog.apply` fires its
automatic transitions without these checks until a member re-registers it. A
manual `schedules.run` fires as the caller, who must be able to `list` every
scanned type. Operating schedules:
[schedules-and-org-admin.md](schedules-and-org-admin.md).

## `target` — scoping an external write

For an action an external principal invokes, `target` tells the external write
gate which row a grant's `row_filter` is checked against, when that row is not
`payload.id`: `{ object_type, by }` with `by` = `"id"`, `{ "input_key":
"<field>" }`, or `{ "actor_attr": "<attr>" }` (the server-derived actor
attribute, never client-supplied).

```json
{
  "name": "guest.set_standing_line",
  "description": "Guest changes their own standing line.",
  "object_type": "guest",
  "input": { "guest_id": { "type": "string" }, "line": { "type": "string" } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent", "service_account"] },
  "idempotency": "none",
  "handler": {
    "steps": [
      { "op": "update", "object": "guest", "where": { "id": "$actor.guest_id" }, "set": { "standing_line": "$input.line" } }
    ]
  },
  "target": { "object_type": "guest", "by": { "actor_attr": "guest_id" } }
}
```

- With `actor_attr`, every `update`/`delete` on the target type must key
  `where: { "id": "$actor.<attr>" }`; anything else (an `$input`, a literal, a
  `$step` ref, no `id`) is `insecure_actor_attr_target`.
- `target` enforces nothing by itself: the gate is skipped for a grant without a
  `row_filter`. Hosted-app publish therefore refuses an `access_manifest` grant
  on a target-declared action without a `row_filter`, and one whose filter never
  binds the caller (literals only, like `deleted = 0`). Bind the caller:
  `id = :actor.guest_id`, optionally `AND deleted = 0`. Want an unconditional
  external write? Don't declare `target`.
- Never write a scope column (`customer_id`, an org column) from `$input` in
  such an action; derive it from `$principal.*` or a `read` of the target row.

## `idempotency`

| Posture | Behaviour |
|---|---|
| `none` | Every call runs. Right for transitions, where the state guard is the retry barrier. A supplied `idempotency_key` is ignored (schedules key every fire, so rejecting it would break them). |
| `per-key` | The caller's `idempotency_key`, scoped to (action, principal), owns an atomic receipt: one caller executes; a same-key retry **replays the first envelope verbatim**; the same key with different input is `CONFLICT` / `IDEMPOTENCY_KEY_REUSED`; a retry while the first is running is `CONFLICT` / `IDEMPOTENCY_IN_PROGRESS`. Replays are kept for **7 days** — a same-key retry after that will **re-execute** the action, not replay it. Use for destructive operations a caller retries. |
| `natural-key` | No caller-key mechanism: the key is ignored as for `none`, and dedup is whatever lookup-before-insert the handler does — racy without a UNIQUE constraint behind it. |

Weaker windows, stated plainly:

- **Receipt settled after commit.** `catalog.apply` (own short transactions per
  sub-step) and the single-statement fast path every plain generated
  create/update/delete uses commit the mutation first and settle the receipt
  after. A crash in between can let a later same-key retry run the mutation a
  second time; for `catalog.apply` concurrent same-key attempts are
  best-effort per sub-step.
- **External effects are at-least-once.** An email or webhook that left before
  the receipt settled is sent again on retry. For an effect the outbox re-invokes
  (a hook, an `on_fail`, a reaction or schedule fan-out), the drain passes the
  row's stable id as `idempotency_key` (an action opts in by declaring that
  input). `email.send_template` reuses it as the MIME `Message-ID` (the mail
  relay has no dedup); `forms.webhook.deliver` sends it as
  `X-Canon-Idempotency-Key` next to `X-Canon-Signature` — dedup on it at the
  receiver if you need exactly-once.

## `transition`

`{ state_machine, from, to | to_expr }`: `from` is `"*"` (any non-terminal
state, and a row whose state is still NULL) or a non-empty array (never matches
NULL); exactly one of `to` / `to_expr`. The marker buys a pre-handler guard
(`INVALID_STATE` outside `from`), the post-handler state write, and the
`on_enter`/`on_exit` hooks. **No declared handler may write a machine-bound
property** — `VALIDATION/direct_state_write`, marker or not. The moved row is
the action's subject when it is of this type (`<type>_id`, `id`, or a
declared or inferred `subject` — above); with none, registration fails
`no_id_input`. To move a set, drop the id input and name rows with
`transition` steps. The machine may be registered before or after the action;
drift between marker and machine is
`VALIDATION/<drift_code>`. Semantics, hooks and set transitions:
[state-machines.md](state-machines.md).

## `handler` and fragments

`{ steps, returns?, on_fail? }` — [actions-handler-patterns.md](actions-handler-patterns.md).
A block several handlers share is declared once with `fragments.declare` and
run with a `call` step; declare it before its callers (a bundle's `fragments/`
applies before `actions/`).

## `presentation`

How a generated UI labels, groups, confirms and reports the action, and builds
its form. It changes nothing about invocation: `name` stays the only id, and
governance alone decides who may call. All fields optional; an unknown key
(`colour`) fails the decode. **This is the one field reference.**

| Field | Type | Meaning |
|---|---|---|
| `label` | string | Button/menu caption and the form's submit label. Fallback: the humanised name. |
| `description` | string | The one line a person reads under the form title — the only description the app shows. |
| `group` | string | Folds related actions into one toolbar menu (`"Lifecycle"`). |
| `icon` | string | Lucide icon name. |
| `emphasis` | `primary` \| `secondary` | `primary` makes it a visible next step on a record; `secondary` is subdued. |
| `destructive` | boolean | Danger styling; with `confirm`, gates the invoke behind confirmation. Not an authz gate. |
| `confirm` | string | Confirmation prompt before invoking. |
| `success` | string | Toast after a completed invoke — form, row, card or queue row alike. |
| `pending` | string | Copy when the invoke parks for approval. |
| `refresh` | `refresh` \| `navigate` \| `none` | After success: re-read, go to the linked record, or nothing. Default `refresh`. |
| `inputLabels` | `{ <input>: string }` | Field label per input. |
| `inputValueLabels` | `{ <input>: { <token>: string } }` | Label per enum input token. |
| `formFields` | input names | Form field order. Listed inputs in this order; an unlisted **required** input is still appended; an unlisted optional one is left out. |
| `inputDefaults` | `{ <input>: string \| number \| boolean }` | Initial value, still editable; an enum default must be one of its tokens. |
| `inputFilters` | `{ <input>: { <property>: "$input.<other>" \| literal } }` | Scopes an `<x>_id` input's picker to records whose `<property>` matches. `$input.<other>` reads another field of the same form (or the record it opened from); while that is empty the picker widens. |
| `showWhen` | `[{ field, op, value? }]` | The subject-row precondition for offering the action: `op` is `eq` \| `neq` \| `in` (array `value`) \| `present` (no `value`), entries ANDed over the subject record (the forms predicate grammar). An action whose `showWhen` does not hold for a row is not offered for it — on every surface that places it per record. Placement only: the handler still enforces its rule. A `field` the subject type (declared subject, else `object_type`) lacks is refused, `VALIDATION/unknown_show_when_field`. The owning type's `display.actions.<a>.showWhen` is ANDed with it. |

```json
{
  "presentation": {
    "label": "Book a job",
    "description": "Book a bike in for repair.",
    "group": "Lifecycle",
    "formFields": ["customer_id", "bike_id", "title", "priority", "quote"],
    "inputDefaults": { "priority": "normal" },
    "inputFilters": { "bike_id": { "customer_id": "$input.customer_id" } }
  }
}
```

- Every input `formFields` / `inputDefaults` / `inputFilters` mention, and every
  `$input.<name>`, must be declared; otherwise `invalid_presentation`. A filter
  shapes the picker only — a handler that needs the rule still asserts it.
- Without `formFields`, a generated writer follows the type's
  `display.formFields`; other actions use input key order with optionals folded.
  The same order lays out the action's dialog and a record's inline Edit (which
  shows only the editable fields, never read-only facts). A bundle export sorts
  JSON keys, so declare `formFields` when order matters.
- **On a record**, actions sharing a `group` collapse into one menu. The visible
  next steps are the record's legal transitions from its current state, then
  other `emphasis: "primary"` actions — at most three, the filled primary
  button last, beside `⋯`. With neither, the first ungrouped action takes the
  one inline slot. A record offers only unhidden actions whose subject is its
  type: a create-family action (no subject: `job.book`) or one whose subject
  is another type never appears there; a create whose subject is the record's
  type (`job.receive`, subject customer) does, the record pre-bound.
  An action a child table on the same page offers is offered there instead
  (one offer per action per page; the child-table rule:
  [viewspec-block-types.md](../../canonify-viewspecs/resources/viewspec-block-types.md)).
- An ObjectType's `display.actions` overrides `label`/`emphasis`, can hide the
  button, and adds a `showWhen` gate — the only way to condition a generated
  action; `group`, `destructive`, `confirm`, `success` come only from here
  ([object-property-types.md](object-property-types.md)).
- **Words.** Labels may be `@t:` keys. A generated writer
  (`<type>.create`/`update`/`replace`) labels fields from the properties'
  `label`s and enum values from `valueLabels`; a custom action's come from
  `inputLabels` / `inputValueLabels`; each falls back to English humanisation.
  Generated verbs are localized. The ownership table:
  [`canonify-viewspecs`](../../canonify-viewspecs/SKILL.md).
- **Clearing.** An organic `actions.update` that omits `presentation` keeps the
  stored block; `"presentation": {}` clears it; a non-empty block replaces it
  whole. A bundle apply clears a presentation the file omits.

## Re-declaring, updating, describing

`actions.declare` is `per-key`:

| Submission | Outcome |
|---|---|
| same name, same JSON, same `idempotency_key` | cached replay |
| same name, same JSON, no key | no-op success |
| same name, **different** JSON | `INVALID_STATE/declaration_changed` — use `actions.update` |
| name in the registry but no stored row | `INVALID_STATE/duplicate_name` |

`actions.update` takes the same declaration and accepts a change, guarded by
the content hash: pass `expectedHash` (from `canon actions describe` or
`catalog.export`) or `force: true`. Neither → `CONFLICT/HASH_REQUIRED`
(`details.current_hash`); stale → `CONFLICT/STALE_DECLARATION` naming the
drifted keys; both → `VALIDATION/CONTRADICTORY_DECLARATION_INTENT`. There is no
`actions.delete`; disable a generated verb with the type's `defaults`.

`canon actions describe <name> --json` (or `canon <name> --explain`) returns the
declaration with a `cli_example` built from the fixture and `declaration_hash`
(`null` for generated actions). It is CLI-only; over REST/MCP, `actions.list`
carries name, description, input schema, governance, transition, subject,
creates and `declaration_hash`, but not idempotency, fixture or `cli_example`.

## What declare checks, and failures

In order: the declaration's own schema → the input descriptors
(`bad_input_schema`) → the handler type-check against the live registry
(`typecheck_failed`, citing step and field) → no write to a machine-bound
property (`direct_state_write`) → the marker against the machine (drift code in
`error.details.code`). All are `VALIDATION`; a failed store is
`VALIDATION/persist_failed` (retry). On success the declaration is stored and
re-registered on every boot.
</content>
