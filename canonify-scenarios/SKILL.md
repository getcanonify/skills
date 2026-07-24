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

The canonical worked example is
[`tests/scenarios/minimal-crm/session.yaml`](../../../tests/scenarios/minimal-crm/session.yaml):
an agent stands up a CRM from an empty Canon — four tables, five ObjectType
refinements, two StateMachines, seven custom ActionTypes — then runs the full
SMB sales loop and cross-checks audit + outbox. Read it once end to end; it is
the reference every pattern here points back to.

---

## Why scenarios — live verification, not jsdom mocking

A scenario does not mock the runtime, stub the DB, or render a fake DOM. It
boots a **real, file-backed libSQL database** (the empty-canon harness in
[`tests/scenarios/_harness/empty-canon.ts`](../../../tests/scenarios/_harness/empty-canon.ts)),
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
default run dispatches through the in-process CLI; setting
`CANONIFY_SCENARIO_TRANSPORT=rest` replays the exact same document over Canon
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
| **read** | `read:` | A typed read — `objects.get` / `objects.list` / `objects.list_types` / `objects.navigate` / `objects.aggregate` / `sql_read` / `schema_describe`. |
| **check** | `check:` | A live cross-check against `_audit_event` (`audit`), `_outbox` (`outbox`), or arbitrary SQL (`db_state`). |
| **clock** | `clock:` | Advance (`by: 30d`) or set (`to: <ISO>`) the harness `TestClock`. |
| **orchestration** | `orchestration:` | Fire a schedule (`schedules.run` / `schedules.run_all_due`) — harness-only, runs outside an action tx. |
| **tick** | `tick: outbox` | One outbox-drain pass; delivers enqueued effects into the recording sinks. |

Refs glue steps together: `$principals.<name>`, `$<bind>`, `$<bind>.<field>`.
`bind:` on a step captures its result data under that name for later `$`-refs.
The `expect:` matcher grammar is five kinds — `equals`, `matches`, `length`,
`count`, `contains` — composed with structural object recursion, so a deep
mismatch cites the exact path (`[21] envelope.data.customer_id: expected
matches '^cus', got 'rec_abc123'`).

Full detail: [resources/scenario-structure.md](resources/scenario-structure.md).
The authoritative schema is `SessionSchema` (Zod) in
[`tests/scenarios/_harness/runner.ts`](../../../tests/scenarios/_harness/runner.ts);
the matcher grammar lives in
[`tests/scenarios/_harness/assertions.ts`](../../../tests/scenarios/_harness/assertions.ts).

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

Every fragment there is validated by the unit test shipped alongside this
skill — see below.

---

## How to run

Scenarios are `bun:test` files. Each lives next to its `session.yaml` as a
one-line `session.test.ts` that calls `runSession`. Run one:

```sh
bun test tests/scenarios/minimal-crm/session.test.ts
```

Run the whole scenario tree:

```sh
bun test tests/scenarios/
```

Replay every scenario over REST instead of CLI (the conformance gate):

```sh
CANONIFY_SCENARIO_TRANSPORT=rest bun test tests/scenarios/
```

The scenarios shipped in this skill's `resources/` are validated +
end-to-end-run by `scenario-docs.test.ts` in this directory:

```sh
bun test docs/skills/canonify-scenarios/scenario-docs.test.ts
```

The repo-wide `bun run ci` gate (biome + `tsc --noEmit` + `test:packages` +
build) typechecks that test and the whole scenario tree runs under
`test:packages` (`bun test packages/ tests/`).

Full detail — reading the verbatim CLI echo, the `AssertionFailure` path
format, isolating one step, and how the empty-canon DB is booted and torn
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
- **`sql_read` pins the CLI connection read-only.** Under the CLI transport,
  the first `read: sql_read` flips the harness DB to `PRAGMA query_only`. Put
  all write/declare steps *before* your first `sql_read`, or a later write
  trips `SQLITE_READONLY`.
- **Single-principal sessions default the principal.** Omit `principal:` only
  when the session declares exactly one (or one named `agent`); multi-actor
  sessions must name it per step.
- **Hook refs are passthrough.** `$row.<col>` / `$row_before.<col>` in an
  `on_enter` hook resolve at execute time against the post-transition row, not
  against session scope — leave them literal.

---

## Contributing back

When you prove a new slice, contribute the scenario so it becomes a permanent
regression guard:

1. Create `tests/scenarios/<your-slice>/session.yaml` (copy an existing one as
   a starting point).
2. Add the one-line `session.test.ts` that calls
   `runSession(\`${__dirname}/session.yaml\`, { transport: TRANSPORT })`, reading
   `CANONIFY_SCENARIO_TRANSPORT` like the existing scenarios do.
3. Run it both ways — CLI and `CANONIFY_SCENARIO_TRANSPORT=rest` — so it stands
   up to the conformance gate.
4. Keep the YAML the source of truth: any future change to the slice is a YAML
   edit, never a TypeScript edit.

---

## Source-of-truth references

- [`tests/scenarios/minimal-crm/session.yaml`](../../../tests/scenarios/minimal-crm/session.yaml) — the canonical end-to-end scenario.
- [`tests/scenarios/_harness/runner.ts`](../../../tests/scenarios/_harness/runner.ts) — `SessionSchema`, the six step kinds, `runSession`, the `$ref` resolver.
- [`tests/scenarios/_harness/assertions.ts`](../../../tests/scenarios/_harness/assertions.ts) — the matcher grammar + the three `check:` runners.
- [`tests/scenarios/_harness/empty-canon.ts`](../../../tests/scenarios/_harness/empty-canon.ts) — the test-DB boot/teardown harness.
- [`../canonify/SKILL.md`](../canonify/SKILL.md) — how to operate Canonify declaratively (the four declaration verbs the scenarios exercise).
- [Phase 17.5 — Declarative Handlers](../../phases/17.5-declarative-handlers.md) §9 — the session-format spec this harness implements.
