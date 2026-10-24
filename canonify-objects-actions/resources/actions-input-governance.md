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
unresolved subject, below), and `actions.update` returns it beside `updated:
true`. The handler never writes `status`: the transition marker does.

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
| `set<enum>` | an array of distinct members of `enum` | `enum` (required), `min` (fewest members) |
| `file` | a string referencing an uploaded file; with `multiple: true` an array of 1–50 distinct ones (0–50 when optional) | `accept`, `max_bytes`, `multiple` |
| `signature` | a string carrying a captured signature (e-signature flows) | — |
| `array` | a list of objects, each shaped by `items` — a map of field → descriptor of any type but `array`, with its modifiers (a row enum renders like a top-level one, row money keeps its currency) — or a list of enum strings when `items` is `{ "type": "string", "enum": [...] }`; a row's `<x>_id` field gets the reference picker and is checked like a top-level reference; rows render as a table (cards on a phone) | `items` (required), `min` / `max` (rows; at most 500, the default — a larger `max` is `array_max_over_row_cap`) |
<!-- input-types:end -->

Every field also takes `optional: true` (may be omitted) and
`nullable: true` (may be `null`). `format` applies to a `string` only and is
one of `email`, `iso8601`, `date` or `multiline`: `iso8601` is a date-time (no
zone = the org's clock) the handler gets as its UTC instant, `date` is `YYYY-MM-DD`, `multiline` takes
any text and makes the form field a multi-line text area. An enum is a `string` with `enum: [...]` (or the ObjectType's
`{ "type": "enum", "values": [...] }`, stored as the first; `set<enum>` takes `values` too); no `timestamp` input
type (use a `string` with `format: "iso8601"`). A modifier on a type that does
not list it is refused at declare time.

```json
{
  "sample_id": { "type": "string", "prefix": "smp_" },
  "priority":  { "type": "string", "enum": ["standard", "express"] },
  "due_at":    { "type": "string", "format": "iso8601" },
  "fee":       { "type": "money", "currency": "DKK", "min": 0 },
  "notes":     { "type": "string", "optional": true }
}
```

A `file` input's `accept` (MIME types `"application/pdf"`, families
`"image/*"`, extensions `".pdf"`) and `max_bytes` (at most the 25 MB upload
cap) are the [file rules](../../canonify-isomorphism/resources/runtime-contract.md)
a `file` property declares; the form's picker offers only those types and
names the limit. A row's `file` field takes them per row (the refusal names
`entries[1].photo`), never `multiple`. With `multiple: true` each file is
checked alone (the refusal names `photos[1]`) and the form picks several; a
handler binds them as attachments of the row it writes with `attach`
([handler steps](actions-handler-patterns.md#steps)) — never into a `file`
property, which holds one file.

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
action whose id inputs all reference other types (a plain create,
`analysis.request`) has none; declare one to bind a create to its parent
(`analysis.run_now`, subject `client`: [patterns.md](../../canonify-app-authoring/resources/patterns.md)).
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
| `operation` | The policy operation the invoke gate checks (below). There is no `policy` field: who may invoke is an access policy's grant ([access-policies.md](access-policies.md)). |

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
`manage`. Owners and admins hold every operation; a member holds no `create`,
`update`, `delete` or `manage` until a policy grants it. Pin `manage` when an
action's effect is more privileged than its name —
`catalog.refine_object_type` itself declares `manage` so a member cannot bypass
the `catalog.apply` gate — and grant it to the group that may run it
([access-policies.md](access-policies.md#gate-an-action-to-a-group)).

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

`{ state_machine, from, to }`: `from` is `"*"` (any non-terminal
state, and a row whose state is still NULL) or a non-empty array (never matches
NULL); `to` is one state, or [to-cases](state-machines.md#to-cases) chosen per row (`to_expr` is `to_expr_unsupported`). The marker buys a pre-handler guard
(`INVALID_STATE` outside `from`), the post-handler state write, and the
`on_enter`/`on_exit` hooks. **No declared handler may write a machine-bound
property** — `VALIDATION/direct_state_write`, marker or not. The moved row is
the action's subject when it is of this type (`<type>_id`, `id`, or a
declared or inferred `subject` — above); with none, registration fails
`no_id_input`. To move a set — or rows of this type from another type's
record, that record being the `subject` — drop the id input and name rows with
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
| `label` | string | Button/menu caption and the form's submit label (unless `submits`). Fallback: the humanised name. |
| `description` | string | The muted line a person reads under the form title (below the record it acts on) — the only description the app shows. |
| `group`, `icon` | string | Folds related actions into one toolbar menu (`"Lifecycle"`); a Lucide icon name. |
| `emphasis` | `primary` \| `secondary` | `primary` makes it a visible next step on a record; `secondary` is subdued. |
| `destructive` | boolean | Danger styling; with `confirm`, gates the invoke behind confirmation. Not an authz gate. |
| `confirm` | string \| `{ text, when, reads? }` | Asked before invoking; `{input.<key>}` reads the form as filled. Plain text always asks; the object form requires `when` ([one grammar](actions-handler-patterns.md#conditions--one-grammar-for-every-surface), fields and `$`-operands `$input.<name>` / `$subject.<column>` / `$reads.<name>`) and asks only while it holds — `[{ "field": "$input.kg", "op": "gt", "value": "$subject.room_kg" }]`; else a tap fires. `reads: [{ name, aggregate: "sum"\|"count", of: "<type>[.<number property>]", where }]` (`$input`/`$subject` operands, `$or`) are read for the caller at invoke. The server decides `when` (reads or not): while it holds an invoke without `confirmed: true` (REST or hosted-app `{ input, confirmed }` body, MCP arg, CLI `--confirmed`) is `CONFLICT` `confirmation_required` (`details.confirm` the copy, `{input.*}` filled, an enum by its value label; `details.reads`); the UI asks, and an agent reads the question, then re-sends with `confirmed: true` / `--confirmed`. Nothing the platform fires can answer it, so no schedule, `on_fail`, state-machine hook or public form may fire such an action: whichever is declared second is refused (the firing one `target_asks_confirmation`, a form at publish; the action `invalid_presentation`). |
| `success` | string | Toast after a completed invoke, any surface; `{returns.<key>}` / `{input.<key>}` read the call ("Made {returns.made} drafts"), each written as its value: a date in the org's convention, an enum as its label, money in its currency, a record id (`$<binding>.id`, an `<x>_id` input) as the record's title. Unknown key → `unknown_copy_token` (an `@t:` copy: in each locale's text). `pending` (parked for approval) reads `{input.<key>}` only. |
| `refresh` | `refresh` \| `navigate` \| `none` | After success, on every surface: re-read (default); go to the record whose id `returns` carries (`"<t>_id": "$<insert or read binding>.id"`), else the subject (neither is `invalid_presentation`); or nothing. |
| `inputLabels`, `inputValueLabels` | `{ <input>: string }`, `{ <input>: { <token>: string } }` | Field label per input (a row field's as `<array>.<field>`; another key is `invalid_presentation`), and per enum token. The form appends "(optional)" to an optional field's label itself, unless the label already ends with it. A label may read a record path as `inputShowWhen` does — `"f1": "$subject.plan.region.item_1_label"`, one action for every region — and reads as the input's own label while that is blank or unreadable. |
| `formFields` | input names | Form field order. Listed inputs in this order; an unlisted **required** input is still appended; an unlisted optional one is left out. |
| `inputDefaults` | `{ <input>: string \| number \| boolean \| { from, map } }` | Initial value, still editable; an enum default must be one of its tokens. A string may be a form reference (below): `"$subject.priority"` opens on the record's value, `"$today"` a date input on the org's today (a date-time on that day's start; any other input is `today_default_not_a_date`). A row field's (`<array>.<field>`) seeds each added row and may read the record the row picks, `"lines.unit_price": "$row.product_id.price"`, taken each time the pick changes unless typed over. An array input may open with one row per child of the subject, `"lines": { "from": "$subject.lines", "map": { "<row field>": "<child property or id>" } }`: the children the reader reads, at most its `max` (more open none, `child_rows_over_cap`). |
| `pickers` | `{ <input>: { facts } }` | What a reference picker (a row field's as `<array>.<x>_id`) shows under the picked record: properties of its type or `{property}` templates, `["price", "{stock} available"]`, each as its value, read as the reader reads them. |
| `inputFilters` | `{ <input>: <filter> }` | Scopes a reference input's picker — `<x>_id`, or the `via` of a link the action's type declares — (a row field's as `<array>.<x>_id`) to the records a [filter](actions-handler-patterns.md#conditions--one-grammar-for-every-surface) admits — `{ "on_stop": false, "status": { "neq": "closed" } }`; an operand may be a form path, as `inputShowWhen` reads one: `$subject.<path>` the record the form was opened from (the action's subject), `$input.<ref>.<path>` the record another reference input picks (`{ "kind": "$input.appointment_type_id.room_kind" }`), each read as the caller reads it; `$input.<other>` another field of the same form, and while it is empty the subject's `<other>` column. While a reference reads nothing the picker widens, and so does the invoke's check (a picked-record path that is NULL or the caller cannot read): a scope is not a rule — hold one (kit held by category) with an `assert`. The invoke holds each id to the same scope: one outside it is `NOT_FOUND` like a missing one, and one whose scope reads an empty `$input.<other>` is `VALIDATION` naming `<other>` (`details.code: reference_scope_input_missing`). A scoped set of at most 5 records renders inline as choices. Otherwise a form picker lists options on demand — a click, typing, an arrow key or tabbing in opens it, the dialog placing focus there does not; Escape closes the list, not the dialog. Each `<property>` is an own property of the type the picker lists: a path through a to-many link is `link_path_to_many`, any other non-property `invalid_presentation` — assert a rule about children in the handler. |
| `submits` | `[{ label, set, primary? }]` | The form's buttons, at most 3, exactly one `primary` (Enter, shown last): each sets declared top-level inputs to values of their type and submits the same action — "Save draft" `{ "confirm_now": false }` / "Confirm order" `{ "confirm_now": true }` is one action whose `transition` step `when` and to-cases land it. The set inputs leave the form; `success` reads them (`{input.confirm_now}` with `inputValueLabels`). Other transports take them as plain inputs. |
| `showWhen` | `[{ field, op, value? }]` | The subject-row precondition for offering the action, entries ANDed over the subject record, in the [one condition grammar](actions-handler-patterns.md#conditions--one-grammar-for-every-surface) (`any_of` / `all_of` groups ≤ 3 deep). An action whose `showWhen` does not hold for a record is not offered for it on any surface that offers it for that record: the record toolbar, a table row, a board card, a queue row and its peek, and the header of a child table under the record (an action about the parent, such as `analysis.add_charge`, is judged on the parent). On a surface with no record to judge — a table header, a reference picker's "+ Create" — a gated action is not offered. Placement only: the handler still enforces its rule. A `field` the subject type (declared subject, else `object_type`) lacks is refused, `VALIDATION/unknown_show_when_field`. The owning type's `display.actions.<a>.showWhen` is ANDed with it. |
| `preview` | `{ fields?, rows?: { <array input>: [...] }, needs? }` | Live values while the form is filled (line net, VAT, total): property names of the record `refresh: "navigate"` would go to (`fields`) and of the row each `$input.<array>` element inserts (`rows`). `actions.preview { action, input }` runs the handler in a transaction it rolls back and returns them as you would read them, asking no confirm; only a refusal is audited (`preview_refused`). A handler reporting per-item outcomes is previewable undeclared and answers its `outcomes` ([handler patterns](actions-handler-patterns.md#sets--read_many-and-for_each)). Names, never a formula; an unreadable name is `invalid_presentation`. `needs: { "pfi": ["pain_now", "pain_worst"] }` shows a field once those inputs are filled — other blank required inputs read as their floor (0 or `min`, false, the first value, empty; a blank reference, date-time or list with `min` has none, so nothing runs); until then it reads "—". A refusal audits the input as sent. |
| `steps` | `[{ title, inputs }]` | The form in 2+ steps: a step bar, Back / Next (refused while a field the step requires is blank), the last step submits (`submits` too). Each top-level input sits in exactly one step (the subject's and a submit's in none); not with `formFields`. |
| `requiredWhen`, `inputShowWhen` | `{ <optional input>: [conditions] }` | `requiredWhen`: required only while the [conditions](actions-handler-patterns.md#conditions--one-grammar-for-every-surface) over `$input.<name>` hold — `{ "red_flag_action": [{ "field": "$input.red_flags", "op": "in", "value": ["fever"] }] }`: the form marks it then, and an invoke leaving it blank is `VALIDATION` `required_when_blank`, its `field_errors` naming the condition. `inputShowWhen`: asked only while its conditions hold over the form and the records it reads — `$subject.<path>` (the record the form opens from) or `$input.<reference input>.<path>` (the record it picks), `<path>` to-one links then a property: `{ "weight_bearing": [{ "field": "$subject.plan.region.code", "op": "in", "value": ["knee", "hip"] }] }`. Not asked, it is required by nothing and an invoke sending it is `VALIDATION` `hidden_input_sent`; a path the caller cannot read leaves it asked; a condition on an input a submit sets is `input_show_when_submitted`, conditions reading each other round a loop `input_show_when_cycle`. A required boolean is a Yes / No question, unanswered until picked; an optional one a checkbox. |

```json
{
  "presentation": {
    "label": "Request analysis",
    "description": "Queue an analysis of a client's sample.",
    "group": "Lifecycle",
    "formFields": ["client_id", "sample_id", "title", "priority", "fee"],
    "inputDefaults": { "priority": "standard" },
    "inputFilters": { "sample_id": { "client_id": "$input.client_id" } }
  }
}
```

- Every input a form block names, and every `$input.<name>`, must be declared;
  `$subject.<column>` a property of the subject type (no subject, no `$subject`);
  `$row.<ref>.<prop>` a reference field of the row and a property it picks of
  that field's type (and enum); `contains` a `set<enum>` or `json`; else `invalid_presentation`.
- To scope a picker by the record a form opens from, read the record:
  `{ "worker_id": { "skills": { "contains": "$subject.skill" } } }` on an action
  whose subject is a shift lists only workers with the shift's skill.
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
  type: a create-family action (no subject: `analysis.request`) or one whose
  subject is another type never appears there; a create whose subject is the
  record's type (`analysis.run_now`, subject client) does, the record pre-bound.
  An action a child table on the same page offers is offered there instead,
  unless it moves the record's own state (one offer per action per page; the child-table rule:
  [viewspec-block-types.md](../../canonify-viewspecs/resources/viewspec-block-types.md)).
- An ObjectType's `display.actions` is the only way to relabel, hide or gate
  a generated action ([precedence](object-property-types.md#display)).
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
