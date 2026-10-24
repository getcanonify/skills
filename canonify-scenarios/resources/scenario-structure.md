# Scenario structure

What the runner's `SessionSchema` accepts. The schema is strict: an unknown key
on a step fails validation instead of being ignored.

## Document and principals

`name` (required), `description` (optional), `principals` (required map,
e.g. `owner: { kind: user, id: usr_owner, org_id: org_test }`), `steps`
(required, non-empty).

| `kind` | Notes |
|---|---|
| `user`, `agent` | members; optional `roles:` (default `owner`, full access) and `groups:` (what `groups.add_member` gives a member at sign-in) |
| `service_account` | member; explicit `permissions:` only (`roles:` ignored) |
| `external`, `app`, `public` | server-minted non-members. Under `canon catalog test` the harness mints the full principal through the same validation the auth edge uses, with deterministic defaults; set `app_id` / `access_manifest_hash`, `hosted_app_id` / `external_actor_id` / `access_manifest_hash` / `grants`, or `form_id` / `target_type` when a policy depends on them. A live target refuses these kinds before dispatch. |

Sessions run under the same deny-by-default policy as production, so your
`_access_policy` rows (row filters, restricted verbs) really gate each step;
give a principal `roles:` to exercise them.

A step without `principal:` uses the only principal, else the one named
`agent`; otherwise it must name one.

## The six step kinds

**`action:`** — invoke an ActionType.

```yaml
- action: lead.create
  input: { id: ld_X, name: Acme, email: a@acme.example.com }
  principal: owner        # optional (rule above)
  bind: lead              # captures envelope.data
  expect: { data: { status: new } }
```

Object/array values in `input:` travel as JSON (how a nested `handler:` rides
through `actions.declare`). An `expect:` with an `error:` key is **denial
mode**: the action must fail, and the error is matched.

**`read:`** — a typed read. Each verb takes a closed set of `input:` keys; an
unknown key fails at parse time naming the legal set (a stray `where:` gets a
"did you mean `filter:`?" hint).

| Verb | `input:` keys |
|---|---|
| `objects.get` | `type`, `id`, `as_of` |
| `objects.list` | `type`, `filter`, `as_of` |
| `objects.navigate` | `type`, `id`, `link`, `as_of` |
| `objects.aggregate` | `type`, `op`, `property`, `group_by`, `expand`, `measure`, `group_by_bucket`, `series_by`, `window_last`, `filter` |
| `objects.list_types`, `actions.list`, `views.list` | — |
| `views.describe` | `name` |
| `apps.list` | `include_broken` |
| `apps.get` | `slug` |
| `sql_read` | `sql`, `args` |
| `schema_describe` | `name` |

- `views.list` / `views.describe` / `apps.*` go through the public
  `view_specs.*` / `apps.*` actions, so a scenario proves Views and App nav
  without `sql_read`. `views.list` returns `name`, `object_type`, `title`,
  `kind`, `broken`, `broken_reason` (broken Views stay visible);
  `views.describe` returns the raw ViewSpec; `apps.list` returns launcher
  records with ordered `nav`; `apps.get` returns one App or `null`.
- `sql_read` returns `{ rows, columns, row_count }`, lifted to the result
  root: `expect: { rows: [{ n: 1 }] }`.
- `objects.aggregate` carries the same inputs as `canon objects aggregate`:
  `filter` is the full `where` grammar, `window_last: N` needs
  `group_by_bucket`, `expand` / `measure` are passed through to the runtime.
  The grammar: [viewspec-binding-and-sources.md](../../canonify-viewspecs/resources/viewspec-binding-and-sources.md).

**`check:`** — a live cross-check.

```yaml
- check: audit
  where: { action: lead.convert, principal: $principals.owner.id }
  expect: { count: 1, status: success }
```

- `audit` (`_audit_event`): `where` keys `action` (`action_id`), `principal`
  (`principal_id`), `status`. `expect.status` must hold for every matched row;
  the rest matches `{ count, rows }`.
- `outbox` (`_outbox`): `where` keys `action`, `idempotency_key_prefix`, `kind`
  (`effect` | `event`), `status`. `expect` keys `count`, `status` (every row),
  `payload` (the first row's parsed payload), `delivered` (reached a recording
  sink — needs a prior `tick:`).
- `db_state`: `where: { sql, args? }`, read-only; matches `{ rows, columns, row_count }`.

| Step | Shape |
|---|---|
| `clock:` | `clock: advance` + `by: 30d` (`ms\|s\|m\|h\|d`), or `clock: set` + `to: <ISO 8601>` — push rows past a `$now`-relative schedule filter |
| `orchestration:` | `schedules.run` (`input: { name }`) or `schedules.run_all_due`; each per-row action runs in its own transaction |
| `tick: outbox` | one drain pass re-invoking due `_outbox` rows into the recording sinks; `expect` matches `data: { delivered, deadlettered }` |

## `$ref` grammar

| Ref | Resolves to |
|---|---|
| `$principals.<name>[.<field>]` | the principal, or a field (`$principals.owner.id`) |
| `$<bind>[.<field>…]` | a prior step's bound result, walked by path |
| `$<bind>[n].field` | numeric index (alias of `$<bind>.n.field`) |
| `$$foo` | the literal `$foo` |

An unresolved ref fails the step before dispatch, naming the ref, the step and
the names bound so far. Subtrees owned by another DSL pass through untouched:
`actions.declare`'s `handler` (`$input.x`, `$now`, `$<step>.x`),
`state_machines.register` hook templates (`$row.x`, `$row_before.x`), and
`schedules.declare` target templates (`$row.id`).

## `expect:` matchers

Matched structurally: a primitive is deep-equal; an array must have the same
length and matches element by element; an object with exactly one matcher key
applies the matcher; any other object recurses over **only the keys it
names** — so objects, including array elements, are subset matches.

| Matcher | Holds when |
|---|---|
| `{ equals: v }` | deep-equal (use it for an exact array or object) |
| `{ matches: 're' }` | string matches the regex |
| `{ length: n }` | array / string / `{length}` has length n |
| `{ count: n }` | number, array length or `{count}` equals n |
| `{ contains: v }` | substring, or an array element deep-equal to v |

A `json` property arrives parsed on typed reads and action envelopes (only
`sql_read` returns raw text); assert the array/object, not its string. A
mismatch reports the full path: `[21] envelope.data.customer_id: expected
matches '^cus', got "rec_abc123"` — root `envelope` for actions, `result` for
reads, the check kind for checks.
