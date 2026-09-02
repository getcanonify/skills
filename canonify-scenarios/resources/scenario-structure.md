# Scenario structure

The authoritative shape is `SessionSchema` (Zod) in the platform's
scenario runner.
This page describes what that schema accepts. The schema is **strict** — unknown
keys on a step are rejected, so a typo fails loudly rather than silently
no-op-ing.

## Top-level document

```yaml
name: my-slice          # required, non-empty — identifies the session in output
description: >          # optional free text
  What this session proves, end to end.
principals:             # required — a map keyed by friendly name
  agent:
    kind: agent         # 'user' | 'agent' | 'service_account' | 'external' | 'app' | 'public'
    id: agt_alice       # the principal id audit rows are attributed to
    org_id: org_test    # the empty-canon harness pins every principal to org_test
steps:                  # required — a non-empty list of step objects
  - ...
```

A `service_account` principal may also carry `permissions: [<action id>, ...]`.
A `user` or `agent` principal may instead carry `roles: [<role>, ...]` — org
roles evaluated by the same PolicyService a production request goes through
(`service_account` ignores `roles:`; it stays explicit-permissions-only).
Omit `roles:` and the principal defaults to `owner` (full access), so a
session that never mentions `roles:` behaves exactly as before this field
existed. Declare `roles: [admin]` / `[member]` / an app-defined role to run a
step as a restricted identity.

`external`, `app`, and `public` are server-minted non-member kinds. The
ephemeral catalog-test harness mints their complete server-shaped principals
through the shared `parsePrincipal` validation seam. Their server fields are
optional in the session document and receive deterministic defaults; provide
`app_id`/`access_manifest_hash`, `hosted_app_id`/`external_actor_id`/
`access_manifest_hash`/`grants`, or `form_id`/`target_type` when policy behavior
depends on those bindings. Live-target runs reject these kinds before any
network request because only the deployed server may mint them.

**Scenario runs enforce real access control.** A self-booted session (the
common case — a `session.test.ts` that only calls `runSession(...)` with no
custom harness) runs under the same deny-by-default authorization a
production request goes through. If your app declares `_access_policy` rows
(row filters, restricted verb sets), those rows actually gate the reads and
actions your scenario drives — a `row_filter` that matches nothing now
actually empties a read rather than being silently ignored. Give a step's
principal `roles:` naming the policy's subject role to exercise that path.

When a step omits `principal:`, the runner defaults it: if the session declares
exactly one principal it uses that one; otherwise it uses the one named `agent`
if present; otherwise the step must name a principal explicitly.

## The six step kinds

Each step is exactly one kind, detected by its leading key.

### `action:` — invoke an ActionType

```yaml
- action: lead.create
  input:                # object matching the ActionType's input schema
    id: ld_REFERRAL01
    name: Acme Industries
    email: buyer@acme.example.com
  principal: agent      # optional; defaults per the rule above
  bind: alice_lead      # optional; captures envelope.data under this name
  expect:               # optional; matcher tree (see below)
    data:
      id: ld_REFERRAL01
      status: new
```

Object / array values in `input:` travel as JSON literals and are re-parsed by
the CLI — this is how the nested `handler:` payload of `actions.declare` rides
through. An `expect:` block carrying an `error:` key flips the step into
**denial mode**: the action MUST fail and the failure is asserted against the
matcher (see [scenario-patterns.md](scenario-patterns.md)).

### `read:` — typed read

```yaml
- read: objects.navigate     # one of the twelve read kinds below
  input:
    type: customer
    id: $conversion.customer_id
    link: contacts
  expect:
    data: { length: 1 }
```

Read kinds: `objects.get`, `objects.list`, `objects.list_types`,
`objects.navigate`, `objects.aggregate`, `actions.list`, `views.list`,
`views.describe`, `apps.list`, `apps.get`, `sql_read`, `schema_describe`. The
`views.*` and `apps.*` reads use the public `view_specs.*` / `apps.*` ActionTypes,
so scenarios can discover declared Views and App navigation identically over
CLI and REST. `views.list` includes broken Views (with `broken_reason`) so
schema-propagation failures remain observable; `views.describe` returns the
raw ViewSpec body. `apps.list` returns ordered `nav` records and `apps.get`
looks up one App by `slug`.

A `sql_read` result carries `{ rows, columns, row_count }` inside `data`; the
assertion layer lifts those to the result root so you can write
`expect: { rows: [{ n: 1 }] }` directly. `objects.aggregate` takes the same
`type` / `op` / `property` / `group_by` / `expand` / `measure` /
`group_by_bucket` / `series_by` / `window_last` / `filter` input as the real
`objects aggregate` verb. `expand` is a string link path whose final hop is
to-many; `measure` is the structured `{ factors: [...] }` input for a product
sum, with property paths and an optional correlated `lookup` factor. The
runtime owns validation of both fields; the scenario harness only carries them
through. For the §2.g PortionPlan, the derived read and its expected figures
look like this:

```yaml
- read: objects.aggregate
  input:
    type: order_line
    op: sum
    expand: product.components
    group_by: product.components.slot
    filter:
      order.location_id: L1
      order.delivery_date: '2026-09-07'
      order.status: locked
    measure:
      factors:
        - quantity
        - product.components.default_ratio
        - lookup:
            objectType: customer_component_ratio
            match:
              customer_id: order.customer_id
              component_id: product.components.component_id
            property: ratio
            default: 1
  expect:
    data:
      groups:
        - { group: { product.components.slot: hot }, value: 18 }
        - { group: { product.components.slot: salad }, value: 12 }
```

`window_last: N` requests the newest N buckets and requires `group_by_bucket`;
`filter` is a JSON object in the full `where` grammar — equality scalars,
`gt`/`gte`/`lt`/`lte` ranges, `{ in }`, `{ neq }`, relative-date tokens; see
[viewspec-analytics.md](../../canonify-viewspecs/resources/viewspec-analytics.md)
for the read's cap, zero-fill, windowing, and per-surface filter forms.

Each read verb accepts a **closed** set of `input:` keys — an unrecognized key
fails validation at parse time, naming the key and the verb's legal set
(a stray `where:` — the CLI flag name — gets a "did you mean `filter:`?"
hint, since the session DSL nests the same JSON under `filter:`), rather than
being silently dropped by the transport:

| Verb | Legal `input:` keys |
|---|---|
| `objects.get` | `type`, `id` |
| `objects.list` | `type`, `filter` |
| `objects.list_types` | *(none)* |
| `objects.navigate` | `type`, `id`, `link` |
| `objects.aggregate` | `type`, `op`, `property`, `group_by`, `expand`, `measure`, `group_by_bucket`, `series_by`, `window_last`, `filter` |
| `actions.list` | *(none)* |
| `views.list` | *(none)* |
| `views.describe` | `name` |
| `apps.list` | `include_broken` |
| `apps.get` | `slug` |
| `sql_read` | `sql`, `args` |
| `schema_describe` | `name` |

### `check:` — live cross-check

```yaml
- check: audit
  where: { action: lead.convert, principal: $principals.agent.id }
  expect:
    count: 1
    status: success
```

Three check kinds:

- `audit` — queries `_audit_event`. `where` keys: `action` (alias `action_id`),
  `principal` (alias `principal_id`), `status`. `expect.status` asserts *every*
  matched row has that status; the rest of `expect` evaluates against
  `{ count, rows }`.
- `outbox` — queries `_outbox`. `where` keys: `action` (alias `action_id`),
  `idempotency_key_prefix`, `kind` (`effect` | `event`), `status`. Special
  `expect` keys: `count`, `status` (every-row), `payload` (matched against the
  first row's JSON-parsed payload), and `delivered` (asserts the effect reached
  a recording sink — requires a prior `tick:` step).
- `db_state` — `where: { sql, args? }` runs read-only SQL; `expect` evaluates
  against `{ rows, columns, row_count }`.

### `clock:` — control the test clock

```yaml
- clock: advance
  by: 30d          # <int><unit>, unit ∈ ms|s|m|h|d
- clock: set
  to: '2026-06-01T00:00:00.000Z'   # ISO 8601
```

Requires a `TestClock`-backed harness (the default). Used to push rows past a
`$now`-relative schedule `where` filter before firing it.

### `orchestration:` — fire a schedule

```yaml
- orchestration: schedules.run
  input: { name: lead.auto_disqualify_stale }
  expect:
    data: { fired: { count: 1 } }
```

`schedules.run` fires one named schedule; `schedules.run_all_due` fires every
due schedule. This is a **harness-only** step — schedule firing re-invokes many
per-row actions, each in its own tx, so it cannot run inside an outer action
transaction. Customer agents never hit this constraint (cross-action chains go
through outbox / schedules / FSM hooks).

### `tick: outbox` — drain the outbox

```yaml
- tick: outbox
  bind: drain          # optional; captures the DrainResult
  expect:
    data:              # the DrainResult is asserted under `data`
      delivered: 1
      deadlettered: 0
```

One `drainOutbox` pass: claims due `_outbox` rows and re-invokes each row's
ActionType through the runtime, so an enqueued `email.send_template` / `notify`
actually delivers to the harness's recording sinks. After a tick, a
`check: outbox` step can assert `status: done` and `delivered:`.

## The `$ref` grammar

Strings starting with `$` are resolved against the evolving scope before
dispatch (matches the handler DSL grammar):

| Ref | Resolves to |
|---|---|
| `$principals.<name>` | the full principal ref object |
| `$principals.<name>.<field>` | a field on it (e.g. `$principals.agent.id`) |
| `$<bind>` | a prior step's bound result |
| `$<bind>.<field>.<...>` | walks into the bound result |
| `$<bind>[n].field` | numeric bracket indexing, an alias of `$<bind>.n.field` |
| `$$foo` | escapes to the literal `$foo` |

An unresolved ref in a runner-owned position fails the step before the value
reaches the CLI or REST transport. The error names the ref, step, and names
currently bound in the session. This prevents a typo such as `$ords[0].id`
from being forwarded as a literal. Use `$$` for an intentional literal
starting with `$`.

Some input subtrees are explicitly opaque because a downstream DSL owns their
refs: `actions.declare.input.handler` (`$input.x`, `$now`,
`$principal.org_id`, `$<step>.field`), `state_machines.register` hook templates
(`$row.x`, `$row_before.x`), and `schedules.declare` target templates
(`$row.id`). Those refs pass through to their own validator/evaluator; all
other scenario input, `where`, `expect`, SQL arguments, and bound-result paths
use the same strict resolver.

## The `expect:` matcher grammar

An expectation is matched structurally against the actual value:

- A **primitive** (string / number / boolean / null) → strict deep-equal.
- An **array** → same length, then each element matched positionally
  (use a matcher to relax the length itself).
- A **matcher object** (exactly one matcher key) → apply the matcher.
- Any **other object** → walk its keys, recursing into each.

Five matcher keys:

| Matcher | Meaning |
|---|---|
| `{ equals: <v> }` | deep-equal to `<v>` |
| `{ matches: '<regex>' }` | actual is a string matching the regex |
| `{ length: <n> }` | array / string / `{length}` object has length `n` |
| `{ count: <n> }` | a number, array length, or `{count}` object equals `n` |
| `{ contains: <v> }` | string substring, or array element deep-equal to `<v>` |

When an expectation reaches a `json`-typed property through a typed action or
read, assert the parsed array/object rather than its JSON-encoded string. The
canonical [JSON and boolean wire contract](../../canonify-objects-actions/SKILL.md#json-and-boolean-wire-contract)
defines this typed-surface shape; `sql_read` is the deliberate raw-`TEXT`
exception. This is a breaking change for assertions written before
`2026.8.28.1409` that expected the stored string.

**Both an object AND an array element are SUBSET-tolerant (tick 6st).** An
`expect:` object only checks the keys it names — a key present on the actual
value but absent from the expectation is ignored. Since an array element
re-enters the same recursion, this applies one level deeper too: `data: {
groups: [{ group: { created_at: '2026-05' }, value: 8 }] }` passes even when
the actual group also carries `partial: true` (`objects.aggregate`'s marker
for the time bucket that contains "now" — see `canonify-viewspecs`'
`viewspec-analytics.md`, *Comparison layers*). Reach for `{ equals: [...] }`
when you want exact-shape array matching instead (every field on every
element, nothing extra allowed).

A mismatch throws an `AssertionFailure` whose message cites the full path, e.g.
`[21] envelope.data.customer_id: expected matches '^cus', got "rec_abc123"`.
The root label is `envelope` for action steps, `result` for reads, and the
check kind for checks.
