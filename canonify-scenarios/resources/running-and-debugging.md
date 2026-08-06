# Running and debugging scenarios

## The run command

A scenario is a `bun:test` file that calls `runSession`. The per-scenario
TypeScript is one line; everything else lives in the sibling `session.yaml`.
Here is the canonical wrapper (verbatim shape from
[`tests/scenarios/minimal-crm/session.test.ts`](../../../../tests/scenarios/minimal-crm/session.test.ts)):

```ts
import { describe, it } from 'bun:test';
import { runSession } from '../_harness/runner';

const TRANSPORT =
  (process.env.CANONIFY_SCENARIO_TRANSPORT as 'cli' | 'rest' | undefined) ??
  'cli';

describe(`minimal-crm scenario (transport=${TRANSPORT})`, () => {
  it(`runs full SMB sales loop declaratively via ${TRANSPORT}`, async () => {
    await runSession(`${__dirname}/session.yaml`, { transport: TRANSPORT });
  }, 60_000);
});
```

Run one scenario:

```sh
bun test tests/scenarios/minimal-crm/session.test.ts
```

Run the whole scenario tree:

```sh
bun test tests/scenarios/
```

The scenario tree also runs as part of the repo gate: `bun run test:packages`
is `bun test packages/ tests/`, which discovers every `*.test.ts` under
`tests/` — including every `tests/scenarios/*/session.test.ts`. `bun run ci`
runs `test:packages` (plus biome, `tsc --noEmit`, and the build), so a broken
scenario fails CI.

## CLI vs REST transport

`runSession` takes a `transport` option (`'cli'` default, or `'rest'`). The
wrapper above reads it from `CANONIFY_SCENARIO_TRANSPORT` so the **same**
`session.yaml` drives both surfaces:

```sh
# default — in-process CLI dispatch (canon …)
bun test tests/scenarios/

# replay the identical document over Canon REST (POST /v1/actions/<name>, …)
CANONIFY_SCENARIO_TRANSPORT=rest bun test tests/scenarios/
```

A scenario that passes on CLI but fails on REST is a surface-isomorphism
regression. Run both before contributing a scenario. `check:` and `clock:`
steps are transport-invariant — they touch the DB / clock directly — so the
matcher grammar is envelope-structural and id-prefix matchers
(`{ matches: '^cus' }`) tolerate the per-run id drift between modes.

## Isolating a single scenario or step

Isolate one scenario by pointing `bun test` at its file:

```sh
bun test tests/scenarios/invoicing/session.test.ts
```

Filter by test name with `-t` (matches the `describe` / `it` string):

```sh
bun test tests/scenarios/ -t invoicing
```

There is no per-step filter — a session is an ordered narrative and later
steps depend on earlier binds. To bisect a failure, comment out the trailing
steps in the `session.yaml` and re-run; the runner stops at the **first**
failing step and reports its index.

## Reading the output

### The verbatim CLI echo

Before dispatching each step the runner prints the literal command it ran,
prefixed with the step's `[NN]` label. CLI mode prints `$ canon …`; REST mode
prints `$ curl …`. Example from a real run:

```
[36] $ canon sql_read \
      --sql 'SELECT count(*) AS n FROM lead WHERE status = ?' \
      --args '["converted"]'
```

This is copy-paste reproducible against the same CLI, and it tells you exactly
which step the run is on when something hangs or fails.

### The `AssertionFailure` path

When an `expect:` clause does not match, the runner throws an
`AssertionFailure` whose message is load-bearing and grep-able:

```
[21] envelope.data.customer_id: expected matches '^cus', got "rec_abc123"
```

- `[21]` — the 1-based step label.
- `envelope.data.customer_id` — the **full structural path** into the
  expectation tree. The root is `envelope` for action steps, `result` for
  reads, and the check kind (`audit` / `outbox` / `db_state`) for checks.
- the rest — what the matcher wanted vs what it got.

The path tells you precisely which key broke, so you rarely need to add
debug prints.

### Action-level failures

If an `action:` step's envelope carries an `error` (and the step did **not**
declare `expect: { error: … }`), the runner surfaces the structured
RuntimeError:

```
SessionError: [14] action:lead.convert: envelope.error.code=INVALID_STATE — …
```

`SessionError` carries `stepIndex`, `stepLabel`, `stepKind`, and the underlying
`envelopeError` ({ `code`, `message`, `issues` }), so the failure pins the
runtime error code without scraping stderr. Common codes: `VALIDATION` (bad
input — re-check the ActionType's input schema), `FORBIDDEN` (governance),
`INVALID_STATE` (FSM guard), `NOT_FOUND` (unknown action / row).

## Test-DB setup and teardown

Each `runSession` call boots a fresh **empty-canon harness**
([`tests/scenarios/_harness/empty-canon.ts`](../../../../tests/scenarios/_harness/empty-canon.ts)):

1. A file-backed libSQL DB (a tmp file by default).
2. The full Canon migration set applied via `runMigrations` (the
   `migrations` array, or `demoFreeMigrations` for the demo-free base) —
   idempotent, so a reboot picks up any later additions.
3. The slice-1 registry built — the slice-1 `customer` ObjectType + the
   "agent vocabulary" platform Actions, then auto-CRUD fills the rest of the
   customer verbs, then `restoreDeclarations` replays anything already in the
   declaration tables.
4. Per-ObjectType read views created so the typed-read helpers can `SELECT`.
5. `createRuntime` + the CLI app wired against the **same** registry, so an
   in-session `actions.declare` is visible to both surfaces with no restart.

Everything else — every business table, ObjectType, StateMachine, and custom
Action — must be declared by your session steps through the public surface.
That is the platform/application boundary made operational: TypeScript is
platform-only; agents drive the application layer through Actions.

On completion (pass **or** fail) the harness is torn down: the libSQL
connection is closed and the DB file (plus its journal / WAL / shm sidecars) is
unlinked. A failing step tears the harness down *before* re-raising, so test
runners observing the failure never leak DB files.

A **reboot recipe** (simulate a process restart) re-mounts the same DB file
with `reuseExistingDb: true` after a teardown with `keepDbFile: true`;
`restoreDeclarations` re-applies every declaration row so the rebooted registry
matches the pre-restart one. Use this to prove declarations survive a restart.
