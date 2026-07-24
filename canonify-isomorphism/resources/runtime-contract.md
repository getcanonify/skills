# The runtime contract

All three surfaces call into one in-process runtime. The runtime is where
governance, audit, idempotency, and the transaction + outbox frame live —
surfaces never reimplement any of it. This file documents the five entry
points and how each transport maps onto them.

## The five entry points

From AGENTS.md §Runtime contract:

```
invoke(action_id, args, principal)        → envelope
get(object_type, id, principal)           → typed object   (in an envelope)
list(object_type, filter, principal)      → typed list     (in an envelope)
navigate(object, link_name, principal)    → typed object | list (in an envelope)
sql_read(statement, principal)            → rows            (in an envelope)
```

- **`invoke`** — the one write/typed-read path. Every mutation in Canonify
  flows through an ActionType and therefore through `invoke`. It is governed
  (Stage 4 approval gate), audited, atomic (tx + outbox), and idempotent.
- **`get` / `list` / `navigate`** — the primary typed read paths. They return
  typed objects, not raw rows, and emit a lighter audit event.
- **`sql_read`** — the analytical escape hatch. It runs on a **read-only**
  Turso connection over auto-generated per-ObjectType views and emits a light
  audit event. There is no `sql_write`; writes are typed-only.

Every one of these returns the same envelope shape (see
`envelope-shape-and-error-handling.md`).

## How each surface maps on

The generators are pure functions over the Catalog — there are no hand-written
per-action route/command/tool files. Each generator emits the wire form for
the surface and a trivial adapter that decodes the wire form into
`(action_id, args, principal)` and calls the runtime.

| Runtime entry | REST | CLI | MCP |
|---|---|---|---|
| `invoke` | `POST /v1/actions/<name>` body `{ "input": {...} }` | `canon <action.name> --flag value` (per-action sugar) or `canon actions invoke <name> --input '{...}'` | tool `actions.invoke` `{ name, args, principal }` |
| `get` | (objects route) | `canon objects get <type> <id>` | tool `objects.get` `{ object_type, id }` |
| `list` | (objects route) | `canon objects list <type> [--where ...]` | tool `objects.list` `{ object_type, filter }` |
| `navigate` | (objects route) | `canon objects navigate <type> <id> <link>` | tool `objects.navigate` `{ object_type, id, link }` |
| `sql_read` | (sql route) | `canon sql_read --sql "..."` | tool `sql_read` `{ statement }` |
| discovery | `GET /v1/actions` | `canon actions list` | tool `actions.list` |
| introspection | — | `canon schema describe [--object <n>]` / `canon actions describe <name>` | tool `schema_describe` `{ name }` |

The CLI exposes both a dotted, per-action sugar form (`canon customer.create
--name X --email a@b`) and a space-separated meta-tool form (`canon actions
invoke …`, `canon objects get …`). The committed, generated manifests at the
repo root — `openapi.json` (REST), `cli-reference.json` (CLI dotted form), and
`mcp-tools.json` (MCP) — are the authoritative wire schemas **for the ActionType
surface** (`POST /v1/actions/<name>`), regenerated from the real catalog and
diffed by CI on every build. They do NOT describe the typed read routes
(`GET /v1/objects/<type>`, `navigate`, `aggregate` in the table above): those
are adapter-emitted at runtime, so consult a live surface's discovery endpoint
(`canon actions list` / `GET /v1/actions`, `canon schema describe`) for the read
surface rather than a committed schema.

### The `{ input }` wrapper (REST + MCP)

REST drives the action with the wire shape `{ "input": { ...args } }`. The
renderer peels exactly one wrapper layer, so an action whose own input has a
field literally named `input` (e.g. `actions.declare`) round-trips correctly —
posting raw args would let that inner field be mistaken for the wrapper. The
MCP `actions.invoke` tool takes `args` as a sibling of `name` and `principal`.
The CLI translates each `--flag value` pair into the args object: scalars
arrive as strings, and structured values (objects, arrays) travel as JSON
strings that the CLI's `shapeArgs` round-trips via `JSON.parse`.

## The principal travels on every call

Every entry point takes a `principal`. Three kinds arrive from the auth edge:

- `user` — human, authenticated via cookie/bearer (REST) or session token (CLI)
- `agent` — LLM, authenticated via API key with **inherited** org permissions
- `service_account` — system-to-system, API key with **explicit** permissions
  (never inherited)

A fourth kind, `system`, is **server-minted only**. It is the internal
cron-scheduler identity (`SystemPrincipal`, `purpose: 'cron'`), minted inside
the scheduler and never reachable from a client. `parsePrincipal` **rejects**
`kind: 'system'` outright, so it can never enter via `X-Principal-Json`,
`--principal-json`, or `CANONIFY_PRINCIPAL_JSON` — the only path to one is direct
internal construction. You'll see it in audit rows but never set it on a call.

On the **MCP surface specifically**, the principal reaches the runtime through
one of three credentials at the `/mcp/<org-slug>` door — an **API key**, a
**central/agent session**, or an **OAuth access token** — but all three resolve
to the same `{ kind, id, org_id, permissions }` before `invoke` runs. Agents and
service accounts keep their API keys; the OAuth token is the interactive path
(Claude Desktop) and is org-scoped + RFC 8707 resource-bound to the URL's org, so
it can only ever produce a principal for that one org. See the "Reaching MCP"
section of the isomorphism SKILL and `connect-claude-desktop.md`.

A principal is `{ kind, id, org_id, permissions? }`. In production, REST
resolves it from headers/bearer and CLI from its session; in the conformance
gate, all three surfaces pass the fixture's full principal as JSON
(`X-Principal-Json` on REST, `--principal-json` on CLI, the `principal`
argument on MCP) so the `org_id` and `kind` are identical across surfaces —
otherwise envelopes that echo `org_id` (like `customer.create`) would diverge.

## Why the contract is the keystone of isomorphism

Because every surface calls the same five functions and the real work happens
only inside them, two surfaces *cannot* compute different results for the same
input unless one of the trivial adapters mis-parses the wire form or
mis-formats the envelope. That narrow seam is exactly what the conformance gate
exercises, and exactly where a "works on CLI but not REST" bug will be found
(see `debugging-surface-divergence.md`).
