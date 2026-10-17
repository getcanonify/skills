---
name: canonify
description: How to operate Canonify declaratively — the manual served by `canon skill instructions` and `GET /v1/skill/instructions`.
version: 2.0.0
---

# Canonify — the manual

Canonify is a **declarative platform**: you build an app by invoking Actions
that declare tables, ObjectTypes, StateMachines, custom ActionTypes and views.
You never write code. Every call — over CLI, REST or MCP — runs inside one
transactional frame with audit, outbox, idempotency and governance built in,
and returns the same envelope on every surface.

This page is the map. Read it once, then go to the skill that owns the topic.

```sh
canon actions list --json     # every action, platform + yours
canon objects list-types      # every ObjectType
canon schema describe         # overview (--object <type> for one)
```

**These skills cover the Integrated tier** — Canon renders your app in its own
web app from ViewSpecs, for your org's members. (Hosted: Canon serves your
static assets to internal or external users; External: you host the app on the
generated REST + SDK. Neither is covered here.)

## The skills

| Skill | Read it for |
|---|---|
| [`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md) | **Before your first table.** The four obligations every model meets. |
| [`canonify-app-authoring`](../canonify-app-authoring/SKILL.md) | The tutorial: an app from an empty org, the worked example, the bundle lifecycle (export / validate / apply), bilingual apps. |
| [`canonify-objects-actions`](../canonify-objects-actions/SKILL.md) | THE reference for tables, ObjectTypes, links, actions, the handler DSL, state machines, import, schedules, e-signature. |
| [`canonify-viewspecs`](../canonify-viewspecs/SKILL.md) | Screens: the ten blocks, binding, recipes. Starts with a one-page cheatsheet. |
| [`canonify-scenarios`](../canonify-scenarios/SKILL.md) | Prove a slice end to end with `session.yaml` and `canon catalog test`. |
| [`canonify-app-lifecycle`](../canonify-app-lifecycle/SKILL.md) | When to stop adding surface; what 1.0 means. |
| [`canonify-isomorphism`](../canonify-isomorphism/SKILL.md) | REST / CLI / MCP mechanics, connecting an MCP client, debugging a surface divergence. |
| [`canonify-esign-sender`](../canonify-esign-sender/SKILL.md) | A ready e-signature sender surface to apply. |
| [resources/error-codes.md](resources/error-codes.md) | Every error code, sub-code, exit code and diagnostic. |

```sh
canon skills                                  # list the published skills
canon skills install canonify-app-authoring   # vendor one into .agents/skills/
canon skills sync                             # refresh all
canon skill --name canonify-viewspecs resource viewspec-cheatsheet.md
```

Skills install to `.agents/skills/` (`.claude/skills` is only an alias);
`--target <dir>` writes elsewhere. Without `canon`: `npx @canonify/skills
install <name>`.

## Auth, org and CLI rules

- **Log in** with `canon auth login`. An agent uses an org-scoped API key.
- **Pick the org per call.** Every command takes `--org <org-id-or-slug>` (a
  slug resolves against your memberships). Precedence: `--org` >
  `CANONIFY_ORG_ID` > config. `canon use` / `canon org switch` **rewrite the
  shared `~/.canon/config.json`** — under several agents or terminals, use
  `--org` or `CANONIFY_ORG_ID=<org> canon …` instead. An unknown slug or a bare
  `--org` exits 64 and lists your orgs.
- **Invoke** with `canon actions invoke <name> --input <json|@file|->`, or the
  per-action sugar `canon <name> --<field> value` (flags are the input names).
  `--output json` for agents.
- **Hyphen alias:** `canon <group> <verb>` maps to `<group>.<verb>` with
  hyphens → underscores (`canon catalog refine-object-type --input @f.json`,
  `canon view-specs declare`). The dotted form always works too.
- **Ask before guessing:** `canon actions describe <name> --json` (or
  `canon <name> --explain`) returns the full declaration, input schema and a
  ready-to-paste `cli_example`.
- A binary below the platform's floor fails with `CLI_UPGRADE_REQUIRED`: run
  `canon upgrade` ([error-codes.md](resources/error-codes.md)).

## The authoring loop

Every change goes through a declaration Action:

| Action | Declares | Reference |
|---|---|---|
| `schema.propose_migration` | A table. Auto-CRUD adds a default ObjectType and `<t>.create` / `read` / `update` / `delete` / `list`. | [tables-and-import](../canonify-objects-actions/resources/tables-and-import.md) |
| `catalog.refine_object_type` | The typed shape: enums, money, labels, links, display, derived properties. | [object-property-types](../canonify-objects-actions/resources/object-property-types.md) |
| `actions.declare` | A custom ActionType whose handler is a JSON document in the **handler DSL**. | [actions-input-governance](../canonify-objects-actions/resources/actions-input-governance.md) |
| `state_machines.register` | A state machine over an enum property; each transition names its action. | [state-machines](../canonify-objects-actions/resources/state-machines.md) |
| `view_specs.declare` | A screen. | [`canonify-viewspecs`](../canonify-viewspecs/SKILL.md) |

Order in a fresh org: tables → ObjectType refinements → actions → state
machines → views → run the loop. Actions precede the machine because
`state_machines.register` refuses a transition naming an undeclared action,
while an action's marker may name a machine registered later.

Organic calls are how you explore; the durable form is a **bundle** on disk —
`canon catalog export` it, commit it, and `canon catalog validate` / `canon
catalog apply` it (apply orders the declarations for you). See
[`canonify-app-authoring`](../canonify-app-authoring/SKILL.md).

Before the first table, read the four obligations in
[`canonify-domain-modelling`](../canonify-domain-modelling/SKILL.md): reuse the
canonical object, anchor every child, reuse the platform primitive,
effective-date time and snapshot money.

**The handler DSL** is an ordered list of typed steps (`read`, `insert`,
`update`, `for_each`, `assert`, …) glued by refs: `$input.<field>`,
`$<step>.<field>`, `$now`, `$principal.id` / `.org_id` / `.kind`,
`$actor.<attr>`. No SQL, no code. The reference and recipes — including
`lead.convert`, the canonical multi-table transition (read the lead, insert a
customer and a contact, stamp the lead, all in one transaction) — are in
[actions-handler-patterns](../canonify-objects-actions/resources/actions-handler-patterns.md).

**Built-in actions you invoke, never declare:** e-signature
([esign-actions](../canonify-objects-actions/resources/esign-actions.md)),
`objects.import` for bulk and historical rows
([tables-and-import](../canonify-objects-actions/resources/tables-and-import.md)),
schedules, reactions and `org.*`
([schedules-and-org-admin](../canonify-objects-actions/resources/schedules-and-org-admin.md)),
`outbox.inspect` / `outbox.requeue`
([outbox-actions](../canonify-objects-actions/resources/outbox-actions.md)).

Two complete, runnable examples ship in `canonify-app-authoring`: the worked
example bundle with its scenario, and the tutorial replay.

## Three surfaces, one runtime

| Surface | Discover | Invoke |
|---|---|---|
| REST | `GET /v1/actions`, `GET /v1/schema_describe` | `POST /v1/actions/<name>` with `{ "input": {…}, "idempotency_key"?: "…" }`; `Authorization: Bearer <api_key>` or `Session <token>` |
| CLI | `canon actions list`, `canon objects list-types`, `canon schema describe` | `canon actions invoke <name> --input …` or `canon <name> --<field> …` |
| MCP | the `actions.list` tool | the `actions.invoke` tool with `{ name, args }` |

The same call returns the same envelope on all three; you never need to think
about which surface you are on. Details and debugging:
[`canonify-isomorphism`](../canonify-isomorphism/SKILL.md).

## The envelope and the runtime frame

```ts
{
  data,              // the result; null on error
  audit_event_id,    // the audit row of this call ('' on a granted read)
  approval_status,   // 'not_required' | 'pending' | 'approved' | 'rejected' (absent on error)
  knowledge_used,    // knowledge ids cited
  outcome_signals,   // post-call feedback
  error?,            // { code, message, issues, field_errors?, failed_precondition?, details? }
  request_id?, approved_by?, next_cursor?   // when relevant
}
```

Every invoke runs, in order: catalog lookup (unknown → `NOT_FOUND`) →
idempotency replay → governance (deny → `FORBIDDEN`, audited `denied`) → input
decode (`VALIDATION`) → BEGIN → transition guard (`INVALID_STATE`) → handler →
state write → `on_enter` / `on_exit` hooks enqueued → handler outbox rows
committed → COMMIT → audit row (written outside the business transaction, so it
survives a rollback) → idempotency cache. Governance, guards, state writes and
side effects live in this frame, which is why a handler is only the domain
effect.

`ActionDomainError` is how a handler turns a domain failure into a typed
envelope: `VALIDATION`, `NOT_FOUND`, `FORBIDDEN`, `INVALID_STATE` (plus
`CONFLICT` and the reserved `RATE_LIMITED`). Codes, details and exit codes:
[resources/error-codes.md](resources/error-codes.md).

Durable side effects:

- **`_audit_event`** — one row per write call (success, denial or error); reads
  are audited on denial only. Columns: `id` (the `audit_event_id`),
  `action_id`, `principal_id`, `principal_type`, `args`, `result`, `status`,
  `error`, `started_at`, `completed_at`, `knowledge_used`. Each org is its own
  database, so there is no `org_id`. Kept **90 days**.
- **`_outbox`** — follow-up work (emails, webhooks, hooks) enqueued atomically
  with the handler's writes and delivered after commit. Event rows kept **14
  days**.
- **Idempotency** — `none`, `per-key` or `natural-key` per action; the replay
  window and rules are in
  [actions-input-governance](../canonify-objects-actions/resources/actions-input-governance.md).

## Reading the audit trail — `canon audit`

```sh
canon audit [--since 7d] [--action <name>] [--principal <id>] [--status denied]
canon audit <audit_event_id>      # one call, untruncated
```

The trail is the `_audit_event` ObjectType (`canon objects list
_audit_event`, `GET /v1/objects/_audit_event`); `canon audit` compiles its
flags into that read.

- **Role decides what you see.** An admin or owner reads the whole org trail; a
  member (and a read-scope OAuth client) reads only rows its own calls wrote —
  an empty page means "you did nothing", not "the org was idle".
- **System types are fenced from apps.** `_audit_event`, `org_member` and the
  e-signature types live in system tables: an `access_manifest` naming a
  `_`-prefixed type or granting a read of a system type is refused
  (`VALIDATION`), and `resource: {}` does not reach one. A hosted app's end user
  or a public visitor also cannot invoke a platform verb over a system table
  (`org_member.list`), nor an action of yours that reads one, cascades a delete
  into one or names one as a calendar or FK subject, nor one whose `on_fail` or
  hook queues a non-reaction-safe action. Members still run all of it; have a
  portal-facing action return only what the portal needs.
- **`sql_read` keeps the same fence.** Your read policy decides each table a
  statement reaches; a table denied, row-filtered or column-masked for you
  refuses the whole statement (the typed read would narrow instead).
  Platform-owned tables (no ObjectType behind them, columns no ObjectType
  exposes, `sqlite_schema`) also need the owner or admin role; `resource: {}`
  does not substitute. Every `sql_read` is itself audited.
- **Control-plane events share the trail** under the reserved `platform.`
  prefix (key mint/revoke, role changes, org rename/delete/leave/export/restore,
  app and form publish, app token mints): `canon audit --origin
  platform|canon|all` (default `all`, filtered server-side). Sign-in and other
  session events are **not** listed in any mode (sign-out is, as
  `platform.session.revoke`); an empty `--origin platform` page is no evidence
  that nobody signed in — ask a Canonify operator for a sign-in trail.

## Reporting a platform problem — `canon report`

```sh
canon report "<failing-command>" "<error-message>" [--context @logs.txt]
canon report status <id> [--quiet]          # no login; the id is the capability
canon report list [--status open|triaged|in_progress|closed|canceled] [--output json]
```

Every non-zero `canon` exit prints a hint pointing here. `report` files the
report and prints its id. `status` prints `id`, `status`, `resolution`,
`created_at`, `command`, `error_message`, `expected`, `severity`, `source`
(`--quiet`: the status only); exits `0` found, `2` unknown id, `1` network
error. `list` needs `canon auth login` and matches your verified email against
each report (exits `0`, `2` bad `--status`, `4` not logged in); an agent key
without a `user_email` in its CLI config lists nothing, though `status` still
works for any id it holds.

## Source of truth

The live catalog. `canon actions describe <name> --json` returns any action's
full declaration and input schema, and a declaration these skills describe
wrongly is refused by the same schema with the offending path in
`error.issues`. When a skill and the live schema disagree, the schema wins —
file the gap with `canon report`.
