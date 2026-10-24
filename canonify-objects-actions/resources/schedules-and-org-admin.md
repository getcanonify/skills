# Schedules, reactions and organization lifecycle

Platform actions every org has. Exact inputs: `canon actions describe <name>`.
The CLI also takes the spaced form (`canon schedules list`, `canon org export`).

## Schedules

`schedules.declare` registers a cadence-driven scan that invokes an action once
per matching row and answers `{ name, registered, next_fire_at }`. There is no
action that removes a schedule.

- `name` — a lowercase `<a>.<b>` pair.
- `target` — `object_type`, `where` (the handler predicate map; `{}` matches
  every row), `action`, optional `input_template`, `delivery` (`invoke`, the
  default, or `outbox`: a durable effect delivered with retry, which needs a
  `reaction_safe` target) and `batch_limit` (rows per run).
- `input_template` — target input → value. `"$row.<col>"` passes the column
  whole, typed as stored (an enum stays its token). A `{<col>}` inside a string
  renders text — an enum or `set<enum>` reads as its value label ("Due {status}"
  → "Due In review"). A `{…}` aimed at an enum, number or boolean input is
  refused, `input_template_not_text`: pass `$row.<col>` instead.
- `idempotency` — `per_row`, optional `suppression_window` (per-row cooldown;
  suppressed rows count as skipped) and `fire_once` (each row at most once ever).
- `trigger.schedule` — one string:

| Form | Fires |
|---|---|
| `daily at HH:MM`, `weekly on <monday…sunday> at HH:MM` | at that local time on the org's calendar (`org.update { timezone }`, UTC when unset), DST-correct |
| `15m`, `1h`, `1d`, `1w`, `every N minutes`, `every N hours` | on the first sweep after declaring, then each time the interval has elapsed since the last run (`1mo` / `1y` are refused: a month has no fixed length — use `daily at`) |
| a 5-field cron string | accepted, but currently fires about every minute — prefer the forms above |

A daily/weekly schedule that missed occurrences fires **once, late**. A new one
fires only if its latest occurrence is under an hour old; otherwise it waits
for `next_fire_at`.

| Action | Does |
|---|---|
| `schedules.list` | Registered schedules (cadence, target, governance, idempotency); filter by `object_type` / `action`. Internal schedules are hidden unless `include_internal: true`. |
| `schedules.describe` | One schedule plus `declared_by`, `declared_at`, `last_fired_at` (any run counts). Unknown → `NOT_FOUND`. |
| `schedules.history` | Past runs, newest first: `limit` (default 50, max 500), `cursor`, `since` (ISO, or a duration back from now: `30m`, `24h`, `2w`, `1mo`). Kept by name, so it outlives the schedule. |
| `schedules.run` | Fire now. `name` runs one **as the caller**; `all_due: true` (owner only — `admin` gets `OWNER_ROLE_REQUIRED`) fires every schedule, internal ones included, due or not. Needs `manage` on `schedule`. |

`schedules.list`, `.describe` and `.history` and `reactions.list` are `manage`
(owners and admins, or a key granted it): they carry every where clause,
input template, target and message template, so a member is `FORBIDDEN`.

- **Unattended work runs on its declarer's current authority**, re-checked
  before every fire; a demoted or removed declarer's schedule records a
  `FORBIDDEN` run instead ([actions-input-governance.md](actions-input-governance.md)).
- **A manual run is a real run**: it sets `last_fired_at` and restarts the
  cadence. Per-row keys are windowed, so a manual run in the same window as a
  cron fire replays that row's cached result.
- **A manual run needs read access**: the caller must `list`, unfiltered, every
  type the schedule scans (`CRON_SCAN_NOT_READABLE_BY_DECLARER`); each row then
  runs through the target action's own governance.
- **History counts are an oracle**: counts, durations and per-row errors show
  only to a caller who may list every scanned type unfiltered; others get
  `counts_hidden: true`. For an unregistered schedule they are hidden from all.
- Schedules and reactions never reach platform types (`_` tables, platform
  ObjectTypes): `VALIDATION`, `platform_object_type_refused`.

**Reactions** (`reactions.declare` / `reactions.list` / `reactions.delete`)
are lowered onto an internal schedule `reaction::<id>` (1-minute cadence, outbox
delivery, fire-once per row). A reaction's `via` must be a `reaction_safe`
action; its `template` is the message text, rendered like a `{<col>}` string
above. **Automatic state-machine transitions** run as internal schedules
`state_machine::<machine>::auto.<via>`.

## Organization lifecycle — `org.*`

| Action | Effect | Who |
|---|---|---|
| `org.update` | Merge-update `name`, fallback `locale`, IANA `timezone` and `brand` (`logoUrl`, `primaryColor`, `surfaceTint`, `uiFont`, `displayFont`, `colorMode`). The timezone is the org's zone: the app shows timestamps on it, date-time inputs edit on it, `@day` / `@week` / `@month` buckets group on it, and `tz: 'org'` date expressions read it (unset: the reader's browser zone, buckets in UTC). At least one field; omit to keep. Brand fields, `locale`, `timezone` and `demo_now` clear with `null` or `""`; `name` cannot be cleared. Invalid zones and unsafe brand values are refused. `demo_now` pins the org clock to a date-time (no offset = the org's zone) for a seeded demo: every `$now`, `:today`, derived `:now` / `:today` and Today reads it, the app shows a demo-clock badge, created_at and audit stay real; `null` = real time. `canon org update --timezone Europe/Copenhagen` | admin, owner (`demo_now`: owner) |
| `org.export` | Disaster-recovery SQL dump of the whole org database, inline as `dump` (`canon org export --output <file>`); omits the transient outbox and external-signer sign-in tables. Sensitive. Not a definition bundle (that is `catalog.export`). | owner |
| `org.restore` | From an `org.export` dump, atomically empties and refills each table present in both dump and live schema (live columns only); dump DDL and `_` tables are never applied. | owner |
| `org.delete` | Suspends now, hard-deletes after 30 days, restorable meanwhile. `confirm` = the exact display name. Returns `next_org_id`. | owner |
| `org.leave` | The caller leaves (deactivated, sessions end). The last owner cannot (`last_owner`). | any member, as a `user` |

An agent calling `org.delete` or `org.restore` is parked until a principal with
`manage` settles it with `actions.approve` / `actions.reject`.
