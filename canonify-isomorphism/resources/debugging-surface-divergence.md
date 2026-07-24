# Debugging "works on CLI but not REST"

Isomorphism guarantees this shouldn't happen, so when it does the bug is real
and lives in a surface adapter (wire-parsing or envelope-formatting), because
the business logic ran through the one shared runtime. The audit log is the
fastest way to prove that and locate the seam.

## First: rule out an unequal call

Most reported "divergences" are not divergences — the two surfaces were called
differently. Check these before anything else:

- **REST wraps its args.** `POST /v1/actions/<name>` takes `{"input": {…}}`,
  not the bare args object. A missing wrapper reads as a validation error on
  REST and success on CLI, which *looks* like drift.
- **The principal differs.** A CLI call runs as your saved session; a REST call
  runs as whatever the `Authorization` bearer resolves to. Different principal,
  different governance outcome — correctly so. Confirm both calls run as the
  same principal before calling it a bug.
- **The args differ after coercion.** CLI flags arrive as strings; a JSON body
  carries real types. `--count 3` and `{"count": 3}` agree, but `--count 3` and
  `{"count": "3"}` may not.

If the inputs and principal are genuinely identical and the envelopes still
differ, it's a real bug. Continue below.

## Diff the two stored envelopes

Every call — on every surface — writes its envelope to the audit log. Fetching
both and diffing them is the ground truth, because it compares what the
*runtime* produced rather than what each surface printed.

1. **Invoke via CLI and capture the audit id:**
   ```bash
   canon customer.create --name Alice --email alice@example.com \
     --org_id org_test --output json
   # → envelope JSON; copy its "audit_event_id"
   ```

2. **Fetch the full stored envelope for that call:**
   ```bash
   canon audit <audit_event_id> --output json
   ```
   `canon audit <id>` returns the **untruncated** detail row (the list view
   truncates envelopes ≥10KB; the detail view never does), so you see the exact
   envelope the runtime produced for the CLI call.

3. **Do the same over REST**, then fetch its stored envelope:
   ```bash
   curl -X POST $BASE/v1/actions/customer.create \
     -H 'content-type: application/json' \
     -H "authorization: Bearer $KEY" \
     -d '{"input":{"name":"Alice","email":"alice@example.com","org_id":"org_test"}}'
   # → envelope JSON; copy its "audit_event_id", then:
   curl $BASE/v1/audit/<audit_event_id> -H "authorization: Bearer $KEY"
   ```
   `GET /v1/audit/<id>` (the route backing `canon audit <id>`) returns the same
   untruncated detail row.

4. **Diff the two stored envelopes.** Mask the fields that are *supposed* to
   differ between any two calls — `audit_event_id`, the minted row id
   (`cust_…`), and timestamps. Whatever else differs is the bug.

Because both invocations ran through the same `invoke`, any remaining
difference is in the surface adapter's parse/format seam, not in the action
handler. That is a platform bug: report it with both audit ids and the two
envelopes, and it can be pinned in the conformance gate so it never regresses.

## Why this is a short list

The surfaces are generated projections of one catalog, and a CI gate invokes
every ActionType on all three surfaces and asserts the normalized envelopes are
byte-identical — an action whose surfaces disagree cannot merge. So the set of
real divergences is small by construction, and "I called them differently" is
overwhelmingly the more likely explanation. See the main SKILL.md for why the
promise holds.
