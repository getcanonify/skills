# The runtime contract

Every surface adapter decodes its wire form and calls one of five runtime
entry points. Each returns the standard envelope.

```
invoke(action_id, args, principal)       → envelope   governed, audited, transactional, idempotent
get(object_type, id, principal)          → envelope   one typed object
list(object_type, filter, principal)     → envelope   typed list, keyset-paginated (next_cursor)
navigate(object, link_name, principal)   → envelope   follow a declared link
sql_read(statement, principal, args?)    → envelope   read-only analytical SQL
```

- **`invoke`** is the only write path: every mutation is an ActionType.
- **`get` / `list` / `navigate`** return typed objects, not raw rows. Reads
  are audited on denial only.
- **`sql_read`** runs on a read-only connection over per-ObjectType views and
  audits every call. The typed reads' policy decides every table a statement
  reaches; where a typed read would filter or mask, `sql_read` refuses the
  statement. `args` is a positional array of null, string, finite number or
  boolean (bound as 1/0); anything else is `VALIDATION`. There is no
  `sql_write`.

Beyond the five base fields, an envelope may carry `request_id` (an approval
request was opened), `approved_by` (`{ kind, id }`, after
`actions.approve` / `actions.reject`), `next_cursor` (list pagination) and
`knowledge` (on `objects.get`).

## Surface mapping

| Entry | REST | CLI | MCP tool |
|---|---|---|---|
| `invoke` | `POST /v1/actions/<name>` `{ "input": {…} }` | `canon <name> --<field> v` or `canon actions invoke <name> --input '{…}'` | `actions.invoke` `{ name, args }` |
| `get` | `GET /v1/objects/<type>/<id>` | `canon objects get <type> <id>` | `objects.get` `{ object_type, id }` |
| `list` | `GET /v1/objects/<type>` | `canon objects list <type> [--where …]` | `objects.list` `{ object_type, filter }` |
| `navigate` | `GET /v1/objects/<type>/<id>/links/<link>` | `canon objects navigate <type> <id> <link>` | `objects.navigate` `{ object_type, id, link }` |
| attachments | `GET /v1/objects/<type>/<id>/attachments` | `canon objects attachments <type> <id>` | `objects.attachments` `{ ref_table, ref_id }` |
| `sql_read` | `POST /v1/sql_read` `{ "sql", "args" }` | `canon sql_read --sql "…" [--args '[…]']` | `sql_read` `{ statement, args? }` |
| discovery | `GET /v1/actions` | `canon actions list` | `actions.list` |
| introspection | — | `canon schema describe [--object <n>]`, `canon actions describe <name>` | `schema_describe` `{ name }` |

Attachments are metadata rows, newest first (MCP sends no bytes), gated by the
parent row: a caller who cannot `get` the row gets that read's envelope with
`data: []`. CLI flag values arrive as strings; objects and arrays travel as
JSON strings and are parsed back.

**Files.** Two REST upload routes (the CLI has none), each `multipart/form-data`
with one part named `file` whose filename and type are kept, each answering
`{ id, content_type, size }` with a `file_…` id:

- `POST /v1/files` stores an unbound file, which only you (its uploader) can
  download. Write its id to a `file` property and every write path binds it to
  that row and property — the generated create and update, a handler `insert` /
  `update` step, `objects.import`. From then on it is read by whoever may read
  that row and property, and listed with the record's attachments. When an
  update moves the property on (another file, or null) the old file is unbound
  again — its uploader's alone, no longer listed. Only the uploader's own write
  binds a file, and an id already bound stays with its first row: storing
  someone else's file id binds nothing.
- `POST /v1/objects/<type>/<id>/files` attaches it to that record (refused
  `FORBIDDEN` / `NOT_FOUND` like a `get` of the record); list them with the
  attachments row above, remove one with `files.detach { file_id }`. Attaching
  and detaching are writes to the record: both need what its `update` needs
  (the grant, its row filter, any restrictive policy), else `FORBIDDEN`. Detach
  refuses (`CONFLICT`) a file a `file` property of its record holds — clear or
  replace the property with an update instead — and an unbound file is
  detached only by its uploader.

Every upload is held to the platform cap (25 MB, `PAYLOAD_TOO_LARGE` / 413). A
`file` property, a `file` action input (a field of an `array` input's rows
too) and an ObjectType's `attachments` may declare `accept` and `max_bytes`;
name the slot in the `POST /v1/files` form (`object_type` + `property`, or
`action` + `input`) and a file it refuses is never stored — another type is
`VALIDATION` (400, `details.rule: "accept"`), a larger file `PAYLOAD_TOO_LARGE`
(413, `details.rule: "max_bytes"`). The write or invoke that stores the id
checks the same rules whatever path uploaded it; the attach route, a handler
step's `attach` and `objects.import`'s attachment history hold every
attachment to its type's `attachments` rules. A file a handler `attach`es is
listed with the slot it was attached under (`ref_property`). The attachments read takes
`include=<link>[,<link>]` (`?include=` over REST, `--include` on the CLI,
`include: [...]` over MCP) to list the files of the records across those
links too — to-many children or a to-one parent — each through its own read
gates, so a linked row or file column you cannot read lists nothing; at most
the first 50 linked rows per link, and `truncated: { links, limit }` names a
link that has more. `objects.import` sets attachment
history (who uploaded a file, when): see
[tables-and-import](../../canonify-objects-actions/resources/tables-and-import.md).

`GET /v1/files/<id>` downloads with the caller's credentials, gated by the row
the file is bound to and the property that holds it, and `HEAD` answers the same headers.
`Content-Disposition` carries the filename; PNG, JPEG, GIF, WebP and PDF are
served inline unless `?download=1`, anything else as a download. A browser
`<img src>` or `<a href>` cannot send a bearer: `GET /v1/files/<id>/link`
returns `{ id, name, mime_type, size_bytes, ticket, expires_at }`, and
`GET /v1/files/<id>?ticket=<ticket>` then serves the file with no header for
10–15 minutes (reusable, not single-use).

**Filtering a REST read.** `GET /v1/objects/<type>`, `…/search` and
`…/aggregate` treat every query param that is not one of their own options as
an equality filter on the property it names (`?status=open&priority=rush`; a
repeated param is set membership) and merge `?where=<json>` over them. Values
are typed by the property (`?is_overdue=true` matches a boolean, derived ones
included). A param naming no property is refused with `VALIDATION`
(`unknown filter property "<name>"`). The list's own options are `id`, `where`,
`sort`, `limit`, `cursor`, `link_titles`, `traversals`, `as_of`; search's are
`q`, `limit`, `offset`, `link_titles`, `traversals` (so a hit carries the same
cells as a list row; CLI `canon objects search <type> <term> --link-titles
--traversals a.b,c.count`, MCP `objects.search` `link_titles` / `traversals`).
The CLI takes `--where` as a JSON object (`--where '{"number":1456}'`, the
only form for operators like `{"total":{"gte":1000}}`) or as the same bare
pairs REST reads (`--where number=1456,status=open`, a repeated key is set
membership); MCP takes an explicit `filter`; all three return the same rows.

## The REST body

```json
{ "input": { "lead_id": "lea_01H…" }, "idempotency_key": "convert-lea_01H…" }
```

`input` is the action's arguments (`canon actions describe <name>` shows the
schema); `idempotency_key` and `knowledge_used` (Knowledge ids the call cites)
are optional. If the body has a top-level `input` key it is this wrapper and
one layer is peeled; otherwise the whole body is the input. A bare body
therefore cannot carry an `idempotency_key`, and breaks any action whose own
input has a field named `input` (`actions.declare` does). Always send the
wrapper.

A `POST /v1/actions/<name>` that changed the catalog (`catalog.*`, `schema.*`,
`view_specs.declare`, `state_machines.register`, a refine, …) answers with
`X-Catalog-Invalidate: catalog`, telling a caching client (the CLI caches the
catalog ~15 minutes) to re-fetch before the next `describe` / `list`.

## NDJSON progress for a long `invoke`

A long call (a large `catalog.apply`) can outlive an edge proxy's idle limit
(~100 s), which kills the connection while the server finishes — and a client
that re-runs a succeeded apply risks the overwrite `baseline` guards against.
Send `X-Canonify-Progress: ndjson`: the response becomes
`200 application/x-ndjson`, flushes at once, heartbeats every few seconds and
ends with exactly one `result` line carrying the real status and envelope:

```
{"type":"progress","elapsed_ms":0}
{"type":"progress","elapsed_ms":5000}
{"type":"result","status":200,"body":{"data":{"applied":true,...},...}}
```

Ignore line types you don't know. Without the header you get the normal single
envelope. `canon catalog apply` opts in automatically (heartbeats on stderr).

## Principals

Every entry point takes a principal `{ kind, id, org_id, permissions? }`,
resolved server-side from the credential:

| Kind | Credential | Authority |
|---|---|---|
| `user` | session (CLI login, web, OAuth on MCP) | the member's org role |
| `agent` | API key | the creator's org role and groups, or the `--role member\|admin` (and `--groups`) the key was minted with (frozen on the key, never re-widened, also across rotation; `owner` is not grantable) |
| `service_account` | API key | explicit permissions only |
| `system` | none — server-minted cron identity | appears in audit rows; any client-supplied `kind: system` is rejected |

Production MCP binds the server-resolved principal; its tools publish no
`principal` parameter, so there is nothing on the wire to forge.
