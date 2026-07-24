# Scenario structure

The authoritative shape is `SessionSchema` (Zod) in
[`tests/scenarios/_harness/runner.ts`](../../../../tests/scenarios/_harness/runner.ts).
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
    kind: agent         # 'user' | 'agent' | 'service_account'
    id: agt_alice       # the principal id audit rows are attributed to
    org_id: org_test    # the empty-canon harness pins every principal to org_test
steps:                  # required — a non-empty list of step objects
  - ...
```

A `service_account` principal may also carry `permissions: [<action id>, ...]`.
`user` and `agent` principals do not.

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
- read: objects.navigate     # one of the seven read kinds below
  input:
    type: customer
    id: $conversion.customer_id
    link: contacts
  expect:
    data: { length: 1 }
```

Read kinds: `objects.get`, `objects.list`, `objects.list_types`,
`objects.navigate`, `objects.aggregate`, `sql_read`, `schema_describe`. A
`sql_read` result carries `{ rows, columns, row_count }` inside `data`; the
assertion layer lifts those to the result root so you can write
`expect: { rows: [{ n: 1 }] }` directly. `objects.aggregate` takes the same
`type` / `op` / `property` / `group_by` / `group_by_bucket` / `series_by` /
`window_last` / `filter` input as the real `objects aggregate` verb
(`window_last: N` requests the newest N buckets and requires `group_by_bucket`;
`filter` is a JSON object in the full `where` grammar — equality scalars,
`gt`/`gte`/`lt`/`lte` ranges, `{ in }`, `{ neq }`, relative-date tokens; see
[viewspec-analytics.md](../../canonify-viewspecs/resources/viewspec-analytics.md)
for the read's cap, zero-fill, windowing, and per-surface filter forms).

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
| `$$foo` | escapes to the literal `$foo` |

Refs whose head is **not** a known binding pass through verbatim — this is how
`actions.declare` carries an opaque handler-DSL payload (`$input.x`, `$now`,
`$principal.org_id`, `$<step>.field`) and `state_machines.register` carries
hook templates (`$row.x`, `$row_before.x`) without you escaping every interior
ref. The session runner only owns refs that address its own scope
(`$principals.*`, `$<bind>.*`).

## The `expect:` matcher grammar

From [`tests/scenarios/_harness/assertions.ts`](../../../../tests/scenarios/_harness/assertions.ts).
An expectation is matched structurally against the actual value:

- A **primitive** (string / number / boolean / null) → strict deep-equal.
- An **array** → element-by-element deep-equal (use a matcher to relax).
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

A mismatch throws an `AssertionFailure` whose message cites the full path, e.g.
`[21] envelope.data.customer_id: expected matches '^cus', got "rec_abc123"`.
The root label is `envelope` for action steps, `result` for reads, and the
check kind for checks.
