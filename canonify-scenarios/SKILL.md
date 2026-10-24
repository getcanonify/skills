---
name: canonify-scenarios
description: How to write and run declarative Canonify scenarios (session.yaml) that prove an app slice end-to-end through the public Canon surface — live verification, not jsdom mocking.
version: 2.0.0
---

# Canonify Scenario Testing

A **scenario** is a `session.yaml` that builds or applies a slice and drives it
step by step through the public surface — the same CLI dispatch or REST routes
a real agent uses. Nothing is mocked: an ephemeral Canon with a real database
runs every step through `runtime.invoke`, so audit, outbox, idempotency,
governance, access policies and the state-machine frame are all live.
Application tests are YAML; you write no test code.

Two full scenarios ship in the `canonify-app-authoring` skill: the worked
example's `scenarios/lab-loop/session.yaml` (a bundle's business loop) and
`resources/tutorial-replay/session.yaml` (an app declared organically, then
driven).

| Page | What |
|---|---|
| [resources/scenario-structure.md](resources/scenario-structure.md) | Document shape, principals, the six step kinds, read verbs, `$ref` and `expect:` grammar |
| [resources/scenario-patterns.md](resources/scenario-patterns.md) | Five complete scenarios: CRM slice, state-machine lifecycle, atomic multi-table action + outbox hook, denials, aggregate reads |

A document is `name`, `principals` and a list of `steps`; each step is one of
six kinds (`action:`, `read:`, `check:`, `clock:`, `orchestration:`,
`tick: outbox`), and `expect:` is a structural matcher tree.

## Run

```sh
canon catalog validate <bundle-dir>                    # static: schema, refs, ViewSpecs; no DB
canon catalog test <bundle-dir> <scenarios-dir>        # ephemeral Canon, apply once, run every scenario
canon catalog test <bundle-dir> <scenarios-dir> --rest # same documents over REST
```

- `canon catalog test` boots an empty, throwaway Canon (your org is never
  touched; it is torn down on pass or fail), runs `catalog.apply` once, then runs every `session.yaml` under
  `<scenarios-dir>` (default: the bundle dir) in alphabetical order —
  `canon catalog test resources/tutorial-replay` runs the organic replay.
- `--rest` (or `CANONIFY_SCENARIO_TRANSPORT=rest`) replays the identical
  documents over REST. Run both: a scenario that passes on CLI and fails on
  REST is a surface bug. `check:` and `clock:` steps are transport-invariant;
  use prefix matchers (`{ matches: '^rec_' }`) for minted ids.
- **One Canon is shared by every scenario in a run** (apply is the slow part).
  An explicit `id:` that an earlier scenario already created collides; the
  failure names the scenario that minted it (`id 'prj_DUP' was already created
  by scenario 'team-a' (step 1, project.create)`). Namespace literal ids per
  scenario (`prj_TEAM_PERSONAL`, not `prj_1`) or omit `id:` and let auto-CRUD
  mint one. `--fresh-per-scenario` boots and applies a new Canon per scenario
  instead, at one apply each.
- Isolate a scenario by pointing `<scenarios-dir>` at its directory. There is
  no per-step filter: the run stops at the first failing step, so bisect by
  commenting out trailing steps.
- Wire `validate` + `test` (both transports) into your repo's CI so every PR
  re-proves the slice; change the slice by editing YAML.

## Read the output

Each step echoes the literal command first — `[NN] $ canon …` (CLI) or
`[NN] $ curl …` (REST) — copy-paste reproducible. A matcher miss is an
`AssertionFailure` with the full path:

```
[21] envelope.data.customer_id: expected matches '^cus', got "rec_abc123"
```

(`[21]` is the 1-based step; the root is `envelope` for actions, `result` for
reads, the check kind for checks.) An action that fails without a declared
`expect: { error: … }` raises a `SessionError` carrying the step index, label,
kind and the envelope's `{ code, message, issues }`
(`[14] action:lead.convert: envelope.error.code=INVALID_STATE — …`). Codes are in the error reference: [../canonify/resources/error-codes.md](../canonify/resources/error-codes.md).

## Gotchas

- **Build state through actions only.** `read: sql_read` and `check: db_state`
  may read; never write through SQL. `sql_read` runs on its own read-only
  connection, so it can sit anywhere among write steps.
- **Declare in dependency order**: tables → refinements → actions → state
  machines → business loop. One statement per `schema.propose_migration`.
- **Declare as a `user`.** `actions.declare`, `catalog.refine_object_type` and
  `view_specs.declare` from an `agent` park for approval and declare nothing.
  Keep an `agent` principal for the business calls.
- **The handler never writes the state column**; the transition marker does.
- **Hook refs are passthrough.** `$row.*` / `$row_before.*` in an `on_enter`
  hook resolve at execute time — leave them literal.
- **Multi-principal sessions name `principal:` on every step.**
- **Use `external` / `app` / `public` principals under `catalog test` only** to
  prove `$principal.kind` guards; a live target refuses them, since a deployed
  org mints them only from its hosted-app, signing or form flows.
