# Access — roles, groups and policies

Who may read which rows and run which actions. Three pieces: the member's
**role** (fixed), the org's **groups** (yours), and **access policies** — the
`policies/<name>.json` files of a bundle. Every gate on every surface reads
the same decision, so a policy written here scopes the CLI, REST, MCP and the
web app alike.

## The default

| Role | May |
|---|---|
| `owner` | everything (the org's provisioned owner policy, an unfiltered grant) |
| `admin` | everything — no grant narrows an admin; a restrictive policy that names them still walls them |
| `member` | `get`, `list`, `navigate`, `aggregate` every type and `invoke` every custom action not declared `"operation": "manage"` (that one only through a group's grant: [Gate an action to a group](#gate-an-action-to-a-group)) — **not** `create`, `update`, `delete` or `manage`; opens every view and app no policy restricts |

Roles are only these three (`canon users` lists members, `canon user role
<id> <role>` changes one). The invoke gate decides an action as
`governance.operation`, else by its name: `<type>.create` / `.update` /
`.delete` are those operations (so a member cannot run the generated writers,
nor a custom action named that way), anything else is `invoke`
([operation](actions-input-governance.md#operation)). `catalog export` and
`catalog validate` are `manage`: the bundle carries every view, app and
policy, so only owners and admins (or a key granted `manage`) read it. So are
the schedule reads (`schedules list` / `describe` / `history`) and
`reactions list`; `connections list` names every connection to a member but
shows its `config` only to who may `manage`.

Policies come in two modes. A **grant** (the default) gives an operation and
says which rows it reaches. A **restrictive policy** (`"mode": "restrict"`)
grants nothing: it is a wall its `operations` are ANDed with. Nothing
denies an operation outright — an operation no grant and no default gives is
refused (so an action declared `manage` is a group's: [gate it](#gate-an-action-to-a-group)) —
except that granting `open` on a view or an app restricts it ([below](#limit-a-view-or-an-app-to-a-group)).

A member's read grants on a type replace the default rows for that type —
that is how a member is scoped to their own rows. A member with no read grant
on the type keeps the default (every row), still behind any wall.

## Groups

Groups are the custom audiences: `finance`, `reviewers`, one per team. A
group is catalog data — `groups/<name>.json` in the bundle, applied with
`canon catalog apply` like a policy:

```json
{
  "name": "finance",
  "managed_by": "managed",
  "source_ref": null,
  "definition": {
    "name": "finance",
    "label": "Finance",
    "description": "Reads every claim in its offices and approves submitted ones."
  }
}
```

The `definition` is `name` (the file's `name` too), and an optional `label`
and `description` — never who is in it: a `members` key is refused
(`group_members_not_catalog`, by `groups.declare` too). Apply
needs an owner or admin, so an owner's agent key can apply it, and then act in
the group through a member key it mints ([below](#see-it-as-a-member)):

```sh
canon catalog apply canon/                      # groups/finance.json + policies/*.json
canon key create --type agent --name fay-demo --role member --groups finance
# a policy with "subject": { "kind": "group", "group": "finance" } now matches that key
```

`canon catalog export` writes each group back as `groups/<name>.json`. A
managed group the bundle no longer declares is pruned — except while a policy
that stays names it (a wall on it would stop walling, a grant would fall back
to the member default) or while anyone is in it; apply then refuses with
`GROUP_STILL_REFERENCED`, writes nothing, and `--force` does not override it.
Drop the group together with the policies that name it, after its members are
removed. A key minted with a pruned group is no longer in it.

People join a group one at a time — membership is never catalog data:

```sh
canon actions invoke groups.add_member --input '{"group":"finance","account_id":"usr_…"}'
canon actions invoke groups.remove_member --input '{"group":"finance","account_id":"usr_…"}'
```

`account_id` is the member's id from `canon users` — the id `:principal.id`
binds. Both are `manage` actions run by a signed-in owner or admin (or a
service account); an agent key is refused them. `groups.declare --input
'{"name":"finance"}'` declares a group outside a bundle (an agent key may run
it; the group is then tenant-owned and never pruned). A policy naming a group
neither the bundle nor the org declares matches nobody (the apply result
warns `undeclared_policy_group`). Sign-in resolves the member's groups onto their
principal (`principal.groups`). To act as a member of a group yourself, mint a
key that carries it ([below](#see-it-as-a-member)).

**An access change holds on the next request**, on every surface: a policy,
group, membership, role or standing change, an ended session, a revoked key;
a row written outside the platform (raw SQL) can take up to 30 seconds. A call whose change cannot be confirmed in force answers `503
ACCESS_CHANGE_UNCONFIRMED`: it is saved and holds within 30 seconds. Two things
outlive a revocation: a file download link (a bearer URL, valid up to 15
minutes) and a live update stream already open (table names only, no rows).


## A policy file

`policies/<name>.json` in the bundle, applied with `canon catalog apply`
like every other artifact (a dropped managed policy is pruned):

```json
{
  "subject": { "kind": "role", "role": "member" },
  "resource": { "object_type": "claim" },
  "operations": ["get", "list", "navigate", "aggregate"],
  "row_filter": "submitted_by = :principal.id"
}
```

That is the file's `definition`; the file is `{ "name", "managed_by":
"managed", "source_ref": null, "definition" }`, as `catalog export` writes it.

| Field | Meaning |
|---|---|
| `subject` | `{ "kind": "role", "role" }`, `{ "kind": "group", "group" }` or `{ "kind": "user", "user": "<id>" }`. Any other kind matches nobody. |
| `resource` | `object_type` (its reads and every action on it) and/or `action` (an action, or a list — any entry matches); naming both requires both. Or `view` / `app` alone (a name or a list): who opens them ([below](#limit-a-view-or-an-app-to-a-group)). `{}` matches everything. |
| `operations` | Any of `get` `list` `navigate` `aggregate` `create` `update` `delete` `invoke` `manage`, or `open` — only on a `view` or `app` (or `{}`), which take nothing else. |
| `mode` | Optional: `"permit"` (the default — a grant) or `"restrict"` (a row wall on one `object_type`, [below](#restrictive-policies--walls); never on a `view` or `app`). |
| `columns` | Optional `{ "mode": "allow" \| "deny" \| "mask", "names": [...] }`: keep only those columns, drop them, or return them as `••••••••` — on every read verb ([below](#a-policy-file)). |
| `row_filter` | Optional string, below. Omit it for an unfiltered grant (`""` is refused). |

Grants combine as a **union**: the rows a principal reaches are the rows any
of its matching grants admits, and a grant without a `row_filter` admits every
row. A member in `finance` with the grant below reads every claim, not only
their own:

```json
{
  "subject": { "kind": "group", "group": "finance" },
  "resource": { "object_type": "claim" },
  "operations": ["get", "list", "navigate", "aggregate"]
}
```

Columns follow the rows: the read verbs share one column set per principal and
type (a mask on `list` masks `get`, `navigate`, `aggregate`). A dropped column is in no row, description or view spec the reader is served (a `confirm.when` over it becomes its text, always asked), so no screen offers it;
a masked one is described `masked`; a type it reads not at all, by name only. Neither is filtered, sorted, grouped, summed
(a tile or chart over it is left out), searched, shown in a link title or
traversal, or read by `sql_read`. Across grants a column is shown when the
grants that show it reach every row the principal reads — an unfiltered grant
showing it, or one per row filter. A column shown only on some rows is withheld
on all of them: an own-rows grant showing `salary` beside an every-row grant
masking it masks `salary` everywhere; an every-row grant showing it shows it
whatever an own-rows grant masks. A file held in a `file` property the caller may not read is left out of
the record's attachments, and its link and download answer `FORBIDDEN`; an
attachment no property holds follows the record.

Apply refuses a policy that does not decode, names an ObjectType, action,
view or app the catalog does not declare, or whose filter does not compile
(`invalid_access_policy`). A list names each entry once and one kind
(actions, views or apps); a list of actions takes no `row_filter` or
`columns`; an action not on `resource.object_type` is refused.

## Restrictive policies — walls

A wall is ANDed on top of whatever the grants and the default give, for every
principal it names — admins included — and no grant widens it. Use one for a
separation that must hold whatever else a member is granted: offices, regions,
clients.

```json
{
  "subject": { "kind": "role", "role": "member" },
  "resource": { "object_type": "claim" },
  "operations": ["get", "list", "navigate", "aggregate"],
  "mode": "restrict",
  "row_filter": "office IN :principal.groups"
}
```

A wall names one `object_type` (never a list), carries a `row_filter` and no `columns`, and
requires `operations`: the verbs it walls, from `get` `list` `navigate` `aggregate` `create`
`update` `delete` (`["get", "list", "navigate", "aggregate"]` above; `["update", "delete"]` walls
only changes). A wall on any read verb walls all four and `create`, `update` and `delete`
too: a row the caller cannot see is a row they cannot change, and the row a
create or update stores must be inside the wall — a member walled to `cph`
cannot move a claim to `aar` or submit one there. `objects.import` is refused
to a walled caller. A wall holds on every path, declared actions included: a
declared action's read, update, delete and transition steps see only the
caller's rows inside it, a step or transition cannot leave a row outside it,
and the children a compound create or replace writes and the rows a delete
cascades to answer to their own type's wall. A wall matches the member's
stored role and groups, so it holds through a read-only OAuth token, and
through an api key the member minted, as well. Schedules, reactions and
state-machine hooks run as the platform, not as the member, and no wall
names them.

**Deny a type outright** with a wall no row passes — the left side is a column (`1 = 0` is refused):

```json
{ "subject": { "kind": "group", "group": "warehouse" }, "resource": { "object_type": "credit_hold" },
  "operations": ["get", "list", "navigate", "aggregate"], "mode": "restrict", "row_filter": "id = 'none'" }
```

To see what a member ends up with, ask the gate itself (owners and admins):
`canon policies effective --account_id usr_… --object_type claim`. It prints, per verb,
whether it is allowed and its row filter (grants unioned, walls ANDed) with the policies
that matched; under `derived_withheld` each derived property the member's reads withhold,
with the type, reason and policies ([below](#derived-properties-under-access)); under
`actions` each declared action's operation, `allowed` and `matched` grants. It reads the
stored policies, a change applied a moment ago included. Pass `--roles` / `--groups`
(JSON arrays) to ask about a membership before you grant it, or for a key: the role and
groups it was minted with.

## Row filters

| Form | Visible rows |
|---|---|
| `<col> = :principal.id` (or `:principal.org_id`, a number, `'text'`) | the column equals it |
| `<col> IN :principal.groups` | the column holds one of the caller's group names |
| `(<filter>) AND (<filter>)` | both hold |
| `{"via_link": "<link>", "where": "<filter>"}` | the row's parent through a belongs-to link (FK on this row) matches — one hop |
| `{"exists": "<child type>", "via_link": "<child's link to this type>", "where": "<filter>"}` | at least one child row matches — one hop |

A filter does not write `OR`: declare one grant per alternative — grants
union.

The two link-through forms are JSON text inside the string:
`"row_filter": "{\"via_link\": \"claim\", \"where\": \"submitted_by = :principal.id\"}"`.
The server ANDs the filter into the read, so a caller's own filter cannot widen
it. The four read verbs share one set of rows per principal and type: the
union of every grant that lists any read verb — counts, totals, search, link
titles and rollups included — even where another grant or the default gives
the verb. How `sql_read` treats a filtered table:
[`canonify`](../../canonify/SKILL.md).

A declared action's `read_many` / `read_aggregate` steps run with the
action's capability, not the invoker's read grants: a member scoped to their
own claims still gets every claim such a step returns. Only a `row_filter` on
the invoke grant itself narrows the steps (to rows of that grant's type), so
scope the step's `where` or gate the action.

## Derived properties under access

A derived property's SQL runs for every reader, so it is shown to a caller
only when that caller reads everything it reads unnarrowed. It is withheld —
absent from the row, refused to aggregates and to filters and sorts (code
`filter_property_withheld` / `sort_property_withheld`) — when its expression names a column the caller's read of the type withholds
(`salary * 12` under a salary mask), reads its own table under a row filter
(a sibling-row subquery), or reads another type the caller's read of which is
row-filtered, walled, refused, or withholds a column it names. A member scoped
to their own `employee` rows does not see `department.headcount`; a member who
reads every employee does. Names are matched by text, so a column named like a
table counts as reading it. `catalog validate` warns `view_reads_withheld_property` per group a view filters or sorts this way for.

To show a parent's field to a row-scoped reader, declare a lookup property
(`claim_line.claim_title` = lookup `claim`, `title`) or use a one-hop link ref
in the view (`claim.title` as a column or queue fact), not a derived property
reading the parent's table: both read the parent under the reader's own grant
on it ([lookups](object-property-types.md#lookup-properties)). A child figure
or flag ("has a note": `exists`) for such a reader is a [rollup](object-links-and-cardinality.md#rollups): it counts only the children they may read.

## Live updates

The live-update stream (`/v1/events/stream`) tells every signed-in member that a row of a
table changed — the table, the kind of change and the time, never the row or its values:
a member scoped away from every row of a type still learns that one changed, and when.

## Hosted-app manifests

A hosted app's `access_manifest` is not a set of policies: its grants AND, because they are
layers of one app's ceiling (a base grant naming columns, a persona branch adding a row filter).
For a member it caps what their policies give; for an external visitor it is their whole
authority. Its read verbs share one row set and one column set per type, like a policy's.

## Gate an action to a group

Declare the action with `"governance": { …, "operation": "manage" }` — no
member holds `manage` by default — and grant it to the group:

```json
{
  "subject": { "kind": "group", "group": "finance" },
  "resource": { "action": "claim.approve" },
  "operations": ["manage"]
}
```

`"action": ["claim.approve", "claim.reject"]` gates both with one policy.
Owners and admins pass regardless. `governance.policy` is gone: a declaration
that sets it is refused with this replacement.

## Limit a view or an app to a group

A policy granting `open` on a view restricts it to that policy's subjects:

```json
{
  "subject": { "kind": "group", "group": "finance" },
  "resource": { "view": "Claim.review" },
  "operations": ["open"]
}
```

`"view": ["Claim.review", "Claim.payouts"]` limits several views with one
policy. A view or app no policy grants `open` on opens for every member, agent and
service account; once one does, only the subjects granted `open` on it (every
such policy adds its subject), owners and admins open it. `{ "app": "<slug>" }`
limits an app the same way. Anyone else:

- does not see it in the app's nav, `GET /v1/schema_describe`'s `views` or
  `view_specs.list`; `view_specs.describe` / `.preview` by its name answer
  `NOT_FOUND` (the web: "This page isn't available") and `apps.get` of a limited
  app `null`, as for a name that does not exist; `apps.list` leaves the app out.
- lands on the app's first nav view they may open when its `defaultView` is
  closed to them; an app with no such view is left out (`apps.get`: `null`).

Hosted-app and form visitors (`external`, `app` and `public` principals) do
not get the "no policy, open to all" rule: their access manifest decides
`open` like any other operation, so a view or app opens for them only where a
manifest grant lists `open` on it (or on `{}`).

**A hidden view is not a data boundary.** Its reads are decided on their
ObjectTypes, so a member who cannot open `Claim.review` still lists every
claim the type grants them — through another view, the CLI or the API. To keep
rows from someone, scope the type: a read grant with a row filter, or a wall.

Apply refuses a view or app policy that names anything else, grants another
operation, carries a `row_filter` or `columns`, or names a view or app the
bundle does not declare; `open` on a type or action; and `"mode": "restrict"`
on a view or app — any matching `open` grant opens it, and nothing walls it
(`invalid_access_policy`).

## What the caller is offered

- The actions list marks each action the invoke gate would refuse the caller
  `invokable: false` with the gate's `refusal: { code: "POLICY_DENIED" |
  "PRINCIPAL_KIND_NOT_ALLOWED", message, details? }` — the message an invoke's
  `FORBIDDEN` would carry (`GET /v1/actions`, MCP `actions.list`, `canon
  actions list`); both absent means they may. The web app offers none of them — record
  toolbar, table row, board card, queue row, child-table header, the picker's
  "+ Create" — so a member without a `create` grant sees no **New**.
- App nav leaves out a view whose ObjectType the caller can neither `get` nor
  `list`, and a view or app limited to others (above); an app with no view
  left is not listed.
- Hiding is placement only: an invoke the gate refuses is still `FORBIDDEN`.

## See it as a member

An owner or admin reads every row, so their own key shows nothing a policy
does. Two ways to watch the policies act on a member:

**Headless** — `canon catalog test <canon> <scenarios>`. A scenario principal
declares `roles: [member]` and `groups: [...]`
([scenario structure](../../canonify-scenarios/resources/scenario-structure.md)),
and each step runs under the production gate. The worked example below is one.

**Live** — mint a key that acts as a member of named groups, and use it:

```sh
canon key create --type agent --name fay-demo --role member --groups finance,cph
CANONIFY_API_KEY=ck_… canon objects list claim     # every Copenhagen claim, as fay
CANONIFY_API_KEY=ck_… canon claim.approve --claim_id clm_…
```

In the browser, exchange the key at `/api/auth/agent/exchange` for a web
session ("See it in the app" in
[`canonify-app-authoring`](../../canonify-app-authoring/SKILL.md)): the session acts as
exactly that key — same rows, same nav, same refusals.

- The key's authority is its minter's, narrowed to the declared role and
  groups. It matches `{group}` grants and walls by its own groups (not its
  minter's) and binds them as `:principal.groups`; a wall over its minter
  still walls it. `:principal.id` is the key's id, so an own-rows filter
  matches the rows the key wrote. `--role member` alone carries no groups.
- Only an owner or admin mints one; each group must already exist
  (`groups/<name>.json` applied, [above](#groups)), and `--groups` needs
  `--role` (a key without a role acts
  as its creator, groups included). The role is capped at the minter's current
  role, and the key acts with nothing once its minter is no longer an owner or
  admin.
- Groups are fixed at mint (mint a new key to change them). The key holds the
  group itself, not its name: once the group is deleted the key is in no group
  of that name, even one re-declared under it. `canon keys` lists each key's role and groups; the
  mint is audited as `platform.key.create`.

## Worked example — expense claims

`access-example/` beside this page: members submit claims and see only their
own claims and claim lines; the `finance` group reads every claim, approves,
and sees the review queue — the app's landing for them; and an office wall
keeps everyone to their own office's claims; a deny-all wall keeps the
`warehouse` group off `credit_hold`. `claim.submit` inserts `submitted_by:
$principal.id`; `claim.approve` is `governance.operation: manage`.

The line policy is the `via_link` form above. The scenario signs in members
as `roles: [member]` with their office as a group, and a finance member with
`groups: [finance, cph]`
([scenario structure](../../canonify-scenarios/resources/scenario-structure.md)),
then proves each claim on this page: own rows on `list` and `aggregate`, the
finance grant unioned with the own-rows grant and walled to Copenhagen, the
generated create refused, approve `FORBIDDEN` outside finance and allowed
inside it; `finance-review` proves the limited view: in finance's nav and
landing, absent from a member's (who lands on `Claim.list`) and `NOT_FOUND`
by name; `deny-all` proves the wall: a warehouse member lists and counts no
credit hold. Run it with `canon catalog test canon scenarios` from that folder;
`canon catalog apply canon` to a real org declares the four groups too, so a
`--role member --groups finance,cph` key acts as that finance member.
