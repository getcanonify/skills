---
title: App patterns — human codes, create-then-transition, bookings
description: Patterns apps keep needing that no single verb names — minting a human-readable code like A-1042, a create action whose new record starts past the state machine's initial state (a 'Run now' on the parent record, a '+ New' in a board column), and bookings (a free slot across practitioner, room and device; a weekly series planned then booked from its dates, with skips, moves and 'this and following'; Today on the org clock; a waitlist whose offers hold their slot and expire forward; Build claims and a partial claim decision).
---

# App patterns

They use the handler DSL ([actions-handler-patterns.md](../../canonify-objects-actions/resources/actions-handler-patterns.md))
and state machines ([state-machines.md](../../canonify-objects-actions/resources/state-machines.md)).

## Minting a human code (`A-1042`)

Store the sequence as an integer column, mint the next number in the create
with `read_aggregate` `max` + 1, and derive the code from it (one source,
never a stored copy):

```json
{ "steps": [
  { "op": "read_aggregate", "object": "analysis", "where": {}, "max": "number", "as": "last" },
  { "op": "insert", "object": "analysis", "as": "analysis", "values": {
    "number": { "op": "+", "left": "$last.max", "right": 1 }, "title": "$input.title" } } ] }
```

On the ObjectType: `"code": { "kind": "derived", "type": "string", "label":
"Analysis", "expression": "'A-' || analysis.number" }`.

- Declare the column `INTEGER UNIQUE` in the DDL, so a collision is refused,
  not duplicated.
- `where: {}` matches every row. `max` over an empty type is `null`, so the
  first mint is refused at RUN time (`VALIDATION`: `operator '+' requires both
  operands to be numbers`; declare cannot flag it): import the first row.

## Create-then-transition — a new record that starts past the initial state

A create cannot write a state-machine status (`direct_state_write`), so a plain
create always lands in the machine's `initial` state. When a record must start
further along — a sample run on arrival, not just queued — insert it and
name it with a `transition` step in the same action, under a `transition` marker
whose `from` is the initial state:

```json
{
  "name": "analysis.run_now", "object_type": "analysis",
  "description": "Run a client's sample now: create the analysis and put it straight on an instrument.",
  "input": { "client_id": { "type": "string", "prefix": "cli_" }, "sample_id": { "type": "string", "prefix": "smp_" }, "title": { "type": "string", "min": 1 } },
  "governance": { "requires_approval": false, "allowed_principals": ["user", "agent"] }, "idempotency": "per-key",
  "transition": { "state_machine": "Analysis.status", "from": ["queued"], "to": "running" },
  "subject": { "param": "client_id", "object_type": "client" },
  "handler": { "steps": [
    { "op": "insert", "object": "analysis", "as": "analysis", "values": { "client_id": "$input.client_id", "sample_id": "$input.sample_id", "title": "$input.title" } },
    { "op": "transition", "object": "analysis", "where": { "id": "$analysis.id" } } ],
    "returns": { "analysis_id": "$analysis.id" } }
}
```

- The transition must be listed in the state machine (`via: analysis.run_now`,
  `from: [queued]`, `to: running`) like any other.
- With `subject` naming the parent (`client_id` → `client`), the client record
  offers it, client pre-bound ("Run now" asks only for the sample and title) —
  in its analyses table's header when the page lists them, once per page.
- On a board it is the "+ New" of its `to` column (`lanes.running.create`; one create per column, [board](../../canonify-viewspecs/resources/viewspec-block-types.md)).

## Bookings — free slots, a weekly series, Today

`booking-example/` beside this page is the whole recipe; its scenario proves each rule on a pinned clock (`canon catalog test canon scenarios` there).

- **Instants, dates and times, never text.** `starts_at` / `ends_at` are
  `timestamp`s (seed them zone-free, `2026-10-23T08:45`: the org's zone reads
  them), a day a `date`, working hours `time`s, picked weekdays a `set<enum>`.
  The schedule is a `calendar` (a day by practitioner, or a week of one): `available` hours, `unavailable` days, room `capacity`, a Show cancelled preset.
- **Free means every rule.** One fragment, `slot_free`, refuses a time outside
  working hours (`date_part` `weekday` / `time` in the org zone against the `time`s),
  off the 15-minute grid (`minute` `%` 15) or held by the practitioner, the room past
  its `capacity` or the device (cancelled rows hold nothing); every booking write calls it:

```json
{ "op": "read_optional", "object": "working_hours", "as": "hours", "where": { "practitioner_id": "$input.practitioner_id",
  "weekday": { "op": "date_part", "part": "weekday", "left": "$input.starts_at" }, "from_time": { "lte": { "op": "date_part", "part": "time", "left": "$input.starts_at" } },
  "to_time": { "gte": { "op": "date_part", "part": "time", "left": "$input.ends_at" } } } }
```

- **A series is planned, then booked from an input array.** `series.plan_weekly` stores a `series_date` for
  exactly `occurrences` `calendar_day` rows (`date`, `weekday`, a running `n`, `starts_at`, `closed`) — `"limit":
  "$input.occurrences"`, weekdays `in` the input, ordered by `n` — whole days with `"tz": "org"` keeping the
  wall-clock time across DST:

```json
{ "op": "date_add", "left": "$input.first_start", "tz": "org",
  "right": { "n": { "op": "-", "left": "$day.n", "right": "$first.n" }, "unit": "d" } }
```

  `series.book` (subject: the series) loops `$input.dates`, opened one row per planned date by `{ "from": "$subject.dates", "map": { "starts_at": "starts_at" } }`,
  with `on_item_fail: "refuse"`: a closed day's assert `skip`s (still counted) and the preview lists each date; a
  busy one offers Move beside Drop. Only an input element can be moved, so never loop a `read_many` there.
  **Skip** cancels an occurrence with a reason (still counted, none added at the end); **Move** sets `starts_at`
  and `moved`, keeping `occurrence_date`; **Cancel this and following** is one `update` by it, so a moved date
  keeps its place: `{ "series_id": "$from.series_id", "occurrence_date": { "gte": "$from.occurrence_date" }, "status": "booked" }`.
- **Today on the org clock.** Derived `:now` and `:today` read it (pin a demo with
  `--demo_now`, never shift seed dates; `date('now')` is UTC real time and is
  refused), so a passed start not checked in is
  Overdue, not Next hour; the view sections it with `rowGroups`:

```json
{ "kind": "derived", "type": "enum", "label": "When", "values": ["overdue", "now", "next_hour", "later", "done"],
  "valueLabels": { "overdue": "Overdue", "now": "Now", "next_hour": "Next hour", "later": "Later", "done": "Done" },
  "expression": "CASE WHEN appointment.status IN ('attended', 'no_show', 'cancelled') THEN 'done' WHEN appointment.status = 'checked_in' THEN 'now' WHEN datetime(appointment.starts_at) <= datetime(:now) THEN 'overdue' WHEN datetime(appointment.starts_at) <= datetime(:now, '+1 hour') THEN 'next_hour' ELSE 'later' END" }
```

  Today's main verb is inline as `{ "action": "appointment.complete", "default": true }` (else the first live verb
  is inline); a list opens on `{ "label": "Waiting", "filter": { "status": "waiting" }, "default": true }` (one preset per list).
- **"If any flag, then X"** is `presentation.requiredWhen` (`inputShowWhen` asks it only then, a region's
  item labels read the plan): the server holds both ([actions-input-governance.md](../../canonify-objects-actions/resources/actions-input-governance.md)):

```json
{ "action_taken": [{ "field": "$input.flags", "op": "present" }] }
```

- **Who reads what.** A practitioner reads the patients they lead or have a live appointment
  with (a grant each; the second an `exists` over appointments whose derived `cover_member_id`
  is null once cancelled), and the claims (their own derived `lead_member_id`), lines and waitlist
  of those they lead (`via_link` to the claim's / patient's); only `front_desk` holds the `manage` decision:

```json
"{\"via_link\": \"patient\", \"where\": \"lead_member_id = :principal.id\"}"
```

  A file kind one group must not see is its own type holding it in a `file` property (an attachment follows its
  record): a `scan` row's `image`, walled from the front desk:

```json
{ "subject": { "kind": "group", "group": "front_desk" }, "resource": { "object_type": "scan" },
  "operations": ["get", "list", "navigate", "aggregate"], "mode": "restrict", "row_filter": "id = 'none'" }
```

  **"Has a note" is a rollup**, `{ "link": "notes", "agg": "exists", "label": "Note written" }`: practitioners read
  notes under a row filter, so a derived `EXISTS` over them is withheld from them and Notes to write's
  `has_note: false` filter refused (`filter_property_withheld`); a rollup counts the notes each reader reads.
- **One offer per slot; expiry advances.** `appointment.offer` lapses the slot's open offer past `expires_at`,
  refuses while one is open or accepted, then `pick_first`s the next waiting entry that fits (`offer_fits`; each
  one passed over is `skipped` with why). A `schedules.declare` (`trigger: { schedule: 15m }`, `idempotency: {
  per_row: true }`) re-runs it over freed slots whose open offer is past `expires_at` — its `target` (a schedule
  scans its action's own type and fires none that asks `confirm`):

```json
{ "object_type": "appointment", "action": "appointment.offer", "input_template": { "appointment_id": "$row.id" },
  "where": { "status": "cancelled", "$exists": { "object_type": "waitlist_offer", "via_link": "slot", "direction": "child",
    "where": { "status": "open", "expires_at": { "lte": "$now" } } } } }
```

- **Where the controls sit.** Add to waitlist's and Plan weekly series' `subject` is the patient: the patient record
  offers them pre-bound. Build claims is the claims board's `createAction` (no subject: the page header and Draft's
  '+'; a board's `actions` only relabels). A type with its own create sets `defaults.create: false`: the generated
  Create leaves every table, board and picker at once, never one `hidden` per table. The record is a `detail`
  declaring its `links` (`tab` / `chip`); undeclared and with no child lists of its own, its two busiest links list in full.
- **A partial decision opens with one row per claim line** the reader reads,
  at its claim: `inputDefaults.lines` is `{ "from": "$subject.lines", "map": {
  "claim_line_id": "id", "approved": "claimed" } }`; the handler approves
  every line as claimed, then each row (at most its claim).
