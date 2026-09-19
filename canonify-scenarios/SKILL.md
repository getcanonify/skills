---
name: canonify-scenarios
description: How to write and run declarative Canonify scenarios (session.yaml) that prove an app slice end-to-end through the public Canon surface — live verification, not jsdom mocking.
version: 1.0.0
---

# Canonify Scenario Testing

A **scenario** is a declarative `session.yaml` document that builds a small
Canon-shaped app and then exercises it, step by step, through the **public
surface** — the same CLI dispatch / REST routes a real agent would use over
the wire. The harness boots an empty Canon org, walks every step, and asserts
each step's envelope against an `expect:` matcher tree. If the scenario passes,
the slice is **provably** working: every mutating step flowed through
`runtime.invoke` exactly as it would in production, with audit, outbox,
idempotency, governance, and the FSM frame all live.

This is how you prove a Canon app slice. You do **not** write TypeScript to
test application behaviour — application engineering is 100% declarative, and so
is its test layer. The per-scenario `.test.ts` is one line: a call to
`runSession(...)`. Everything that varies — principals, steps, assertions —
lives in the YAML.

The canonical worked example is the `minimal-crm` scenario (maintained in
the Canonify platform repo, and excerpted throughout this skill's
resources): an agent stands up a CRM from an empty Canon — four tables,
five ObjectType refinements, two StateMachines, seven custom ActionTypes —
then runs the full SMB sales loop and cross-checks audit + outbox. The
copy-paste fragments in
[resources/scenario-patterns.md](resources/scenario-patterns.md) are the
reference every pattern here points back to.

---

## Why scenarios — live verification, not jsdom mocking

A scenario does not mock the runtime, stub the DB, or render a fake DOM. It
boots a **real, file-backed libSQL database** (the runner's ephemeral
empty-canon harness),
applies the real Canon migrations, builds the real slice-1 registry, and
dispatches every step through the real CLI or REST surface. When the FSM guard
rejects a transition, it is the **actual** Phase 18.5 guard rejecting it. When
an `on_enter` hook enqueues a welcome email, the row lands in the **real**
`_outbox` table and a `tick:` step drains it into the **real** recording sink.

The payoff is that a passing scenario carries no asterisks. There is no "works
in the test but not in production" gap, because the test path *is* the
production path. A mocked unit test can pass while the surfaces disagree; a
scenario cannot. This is the opposite of jsdom-style mocking, where you assert
against a hand-built fake and hope it matches reality.

The same `session.yaml` also drives **every transport identically**. The
default run dispatches through the in-process CLI; passing `--rest` to
`canon catalog test` replays the exact same document over Canon
REST (`POST /v1/actions/<name>`, `GET /v1/objects/*`). A scenario that passes
on CLI but fails on REST is a surface-isomorphism regression, caught
automatically. You never reason about which surface you are on — the envelope
shape is the same on all of them.

---

## Structure in 60 seconds

A `session.yaml` has three top-level keys: `name`, `principals` (a map keyed by
friendly name), and `steps` (a non-empty list). Each step is one of **six
kinds**, detected by its leading key:

| Kind | Leading key | What it does |
|---|---|---|
| **action** | `action:` | Invoke an ActionType through the surface. Carries `input:`, optional `principal:`, `bind:`, `expect:`. |
| **read** | `read:` | A typed read — object reads, Action/View/App discovery, `sql_read`, or `schema_describe`. |
| **check** | `check:` | A live cross-check against `_audit_event` (`audit`), `_outbox` (`outbox`), or arbitrary SQL (`db_state`). |
| **clock** | `clock:` | Advance (`by: 30d`) or set (`to: <ISO>`) the harness `TestClock`. |
| **orchestration** | `orchestration:` | Fire a schedule (`schedules.run` / `schedules.run_all_due`) — harness-only, runs outside an action tx. |
| **tick** | `tick: outbox` | One outbox-drain pass; delivers enqueued effects into the recording sinks. |

Refs glue steps together: `$principals.<name>`, `$<bind>`, and
`$<bind>.<field>` (or `$<bind>[n].field` for numeric indexing).
`bind:` on a step captures its result data under that name for later `$`-refs.
The `expect:` matcher grammar is five kinds — `equals`, `matches`, `length`,
`count`, `contains` — composed with structural object recursion, so a deep
mismatch cites the exact path (`[21] envelope.data.customer_id: expected
matches '^cus', got 'rec_abc123'`).

Full detail: [resources/scenario-structure.md](resources/scenario-structure.md).
The authoritative schema is `SessionSchema` (Zod) in the platform's
scenario runner; the strict parse means an unknown key fails loudly at
validation time rather than silently no-op-ing.

### Principals

`principals.<name>.kind` is a closed enum: `user`, `agent`,
`service_account`, `external`, `app`, or `public`. The first three are member
principals. `service_account` takes explicit `permissions:`, while `user` and
`agent` may take `roles:`; omitted member roles default to `owner` in the
ephemeral production-policy harness.

`external`, `app`, and `public` are server-minted non-member kinds. In the
ephemeral catalog-test harness they are constructed through the same
`parsePrincipal` seam used by the CLI/REST auth boundary, so handlers receive
the real discriminated-union shape rather than a kind-only cast. The harness
fills deterministic values for omitted server fields; provide
`app_id`/`access_manifest_hash`, `hosted_app_id`/`external_actor_id`/
`access_manifest_hash`/`grants`, or `form_id`/`target_type` when a scenario
needs to exercise manifest, actor, app, or form policy behavior.

Use these kinds to prove `$principal.kind` guards in `catalog test`. A live
target refuses them before network dispatch — for example,
`external principals cannot be minted against a live org — run under catalog
test` — because a deployed org must mint them from its hosted-app, signing, or
form flow. Do not replay a client-supplied special principal through the live
REST target.

---

## Object, Action, View, and App discovery

Discovery reads use the same generated ActionType path on CLI and REST, so a
scenario can prove the public catalog without querying `_view_specs` or
`_canon_apps` through `sql_read`:

- `read: objects.list_types` returns declared ObjectType names.
- `read: actions.list` returns the live ActionType names.
- `read: views.list` invokes `view_specs.list` and returns each declared View's
  `name`, `object_type`, `title`, `kind`, `broken`, and `broken_reason`.
  Broken Views remain visible here so propagation failures are observable.
- `read: views.describe` invokes `view_specs.describe` with `name` and returns
  the full raw ViewSpec body.
- `read: apps.list` and `read: apps.get` invoke the corresponding App read
  actions. `apps.list` returns launcher records and ordered `nav`; pass
  `include_broken: true` when degraded Apps should be included. `apps.get`
  takes a `slug` and returns one record (or `null`).

Use these reads after declaration steps when a scenario needs to prove that a
View or App exists and that App navigation points at the intended View.

---

## Patterns

[resources/scenario-patterns.md](resources/scenario-patterns.md) collects
copy-paste-ready, harness-valid fragments for the shapes you will reach for
most:

1. **CRM slice** — declare a table, refine the ObjectType, drive the
   create/qualify/convert loop, navigate links.
2. **Invoice FSM lifecycle** — register a state machine, walk
   draft → sent → paid, assert the denied illegal transition.
3. **Multi-table atomic action** — the `lead.convert` centerpiece: one handler
   that reads + inserts across three tables inside a single tx.
4. **Outbox-enqueue assertion** — an `on_enter` hook enqueues an email; assert
   the `_outbox` row with a `check: outbox` step (and `tick:` + `delivered:`
   to prove it actually delivered).
5. **Governance / FSM-guard denial** — assert a call fails with
   `expect: { error: { code: FORBIDDEN } }` or `INVALID_STATE`, then prove the
   denied call did not mutate the row.

Every fragment there is validated and end-to-end-run against the real
harness in the platform's CI, so a copy-paste starts from a known-green
state.

---

## How to run

Scenarios live in your app repo as plain `session.yaml` files in a
scenarios directory, next to your exported catalog bundle. The runner is
the `canon` CLI itself:

```sh
canon catalog validate <bundle-dir>          # static pass — schema, refs, ViewSpecs
canon catalog test <bundle-dir> <scenarios-dir>   # boot an ephemeral Canon, apply once, run every scenario
```

`canon catalog test` boots an **ephemeral, empty Canon**, runs
`catalog.apply` on your bundle once, then discovers and runs every
`session.yaml` under `<scenarios-dir>` against it (alphabetical order).
Nothing touches your real org. This is the CI gate a Canon-app repo runs
on every PR.

Replay every scenario over REST instead of the in-process CLI (the
conformance angle — same documents, second transport):

```sh
canon catalog test <bundle-dir> <scenarios-dir> --rest
```

`--fresh-per-scenario` boots + applies a brand-new Canon for every
scenario instead of sharing one across the run (see Gotchas below).

Full detail — reading the verbatim CLI echo, the `AssertionFailure` path
format, isolating a failure, and how the ephemeral DB is booted and torn
down — is in
[resources/running-and-debugging.md](resources/running-and-debugging.md).

---

## Gotchas

- **No raw SQL writes.** A scenario builds state through ActionTypes only.
  Reads may use `read: sql_read` / `check: db_state`, but you never mutate via
  SQL — that defeats the point of proving the declarative surface.
- **Declare in dependency order.** Tables → ObjectType refinements → custom
  Actions → StateMachines → business loop. A `transition:` marker on an Action
  cross-validates against the SM; declare the Actions *before* the SM that
  names them (the cross-validator resolves both directions).
- **The handler never writes the FSM state column.** The runtime auto-writes
  `<property> = <to>` after a transition handler returns;
  `actions.declare` rejects a handler that writes it directly.
- **`sql_read` is read-only on its own connection; step order is free.**
  `sql_read` runs on a dedicated read connection pinned with
  `PRAGMA query_only` (with a statement-level guard where the driver rejects
  the pragma). The harness DB is never flipped: write and declare steps
  succeed before and after any `sql_read`, in any order. The only rule is
  that a `sql_read` statement itself cannot write.
- **Single-principal sessions default the principal.** Omit `principal:` only
  when the session declares exactly one (or one named `agent`); multi-actor
  sessions must name it per step.
- **Hook refs are passthrough.** `$row.<col>` / `$row_before.<col>` in an
  `on_enter` hook resolve at execute time against the post-transition row, not
  against session scope — leave them literal.
- **Scenario runs enforce real access policies.** A self-booted session runs
  under the same deny-by-default PolicyService a production request goes
  through, so an app's `_access_policy` rows (row filters, restricted verbs)
  actually gate the session. A principal defaults to the `owner` role (full
  access) unless a step's principal declares `roles:` — see
  [scenario-structure.md](resources/scenario-structure.md).
- **`read:` input keys are a closed set per verb.** An unrecognized key
  (e.g. `where:` instead of `filter:`) fails validation at parse time instead
  of being silently ignored by the transport.
- **`canon catalog test` shares ONE Canon across every scenario in the run.**
  Namespace every literal `id:` you write (prefix by scenario) or two
  scenarios can collide; `--fresh-per-scenario` opts out of sharing entirely
  at the cost of one apply per scenario. See
  [resources/running-and-debugging.md](resources/running-and-debugging.md#scenarios-share-a-database-under-canon-catalog-test).

---

## Contributing back

When you prove a new slice, keep the scenario so it becomes a permanent
regression guard:

1. Create `<scenarios-dir>/<your-slice>/session.yaml` in your app repo
   (copy an existing one as a starting point).
2. Run it both ways — `canon catalog test …` and
   `canon catalog test … --rest` — so it stands up to the conformance
   gate.
3. Wire `canon catalog validate` + `canon catalog test` into your repo's
   CI so every PR re-proves the slice.
4. Keep the YAML the source of truth: any future change to the slice is a
   YAML edit, never a code edit.

---

## Source-of-truth references

All of these live in the Canonify platform repo; they are cited here as
evidence of where each behaviour is implemented:

- the `minimal-crm` scenario — the canonical end-to-end session.
- the scenario runner (`SessionSchema`) — the six step kinds, `runSession`, the `$ref` resolver.
- the assertion module — the matcher grammar + the three `check:` runners.
- the empty-canon harness — the ephemeral-DB boot/teardown.
- the apply-and-run loop — what `canon catalog test` drives: one apply, shared-vs-`--fresh-per-scenario` Canon lifetime, the id-collision naming diagnostic.
- [`../canonify/SKILL.md`](../canonify/SKILL.md) — how to operate Canonify declaratively (the four declaration verbs the scenarios exercise).
- Phase 17.5 §9 (platform spec) — the session-format contract this harness implements.
