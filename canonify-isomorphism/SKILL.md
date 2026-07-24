---
name: canonify-isomorphism
description: Why REST, CLI, and MCP are three isomorphic projections of one Canon catalog — the runtime contract they all call, the envelope they all return, why they can't drift, and how to debug 'works on CLI but not REST'. Read SKILL.md first, then deep-dive on the runtime contract, the envelope shape, or how to debug a surface that seems to diverge.
version: 1.0.0
---

# Canonify REST/CLI/MCP Isomorphism

Canonify exposes one Canon — its ObjectTypes, ActionTypes, ViewSpecs, and
Apps — over three transports: a REST API, a `canon` command-line tool, and an
MCP server. The promise of isomorphism is simple and strict: **the same
operation, with the same input, produces the same result no matter which
transport you reach for.** Pick the surface that fits your client, not the one
that happens to support the feature — because every surface supports every
feature, by construction.

This skill explains why that promise holds, what the three surfaces share
underneath, and how to debug the rare case where they appear to disagree.

## Three surfaces, one catalog

The mental model that matters: REST, CLI, and MCP are not three separate
codebases that happen to agree. They are three **generated projections** of a
single source of truth — the Catalog. The Catalog is the typed registry of
ObjectTypes, ActionTypes, principals, bindings, views, and annotations. Each
surface is a pure function over it:

```
src/surfaces/rest/render.ts   →  generateRestApp(catalog)
src/surfaces/cli/render.ts    →  generateCliApp(catalog)
src/surfaces/mcp/render.ts    →  generateMcpServer(catalog)
```

There are no hand-written REST routes, CLI commands, or MCP tool definitions
for ActionTypes. Declare a new ActionType and it auto-appears in all three
surfaces at once. When auto-CRUD scans a freshly migrated table, the new
`create` / `read` / `update` / `delete` / `list` actions become reachable over
REST, CLI, and MCP simultaneously — no restart, no per-surface plumbing. The
only hand-written code per surface is a trivial adapter shell: parse the
principal, parse the args against the ActionType's schema, call the runtime,
format the envelope. AGENTS.md caps each adapter at a few hundred lines, and a
growing adapter is treated as a smell.

This is why "is feature X available on MCP?" is almost always the wrong
question. If X is in the Catalog, it is on every surface. If it isn't, it is on
none.

## The runtime contract

Underneath all three surfaces sits one in-process runtime. Every surface
adapter funnels into exactly five entry points:

```
invoke(action_id, args, principal)   → envelope   (typed, governed, audited write/read)
get(object_type, id, principal)      → envelope   (one typed object)
list(object_type, filter, principal) → envelope   (typed list, keyset-paginated)
navigate(object, link_name, principal) → envelope (traverse a declared link)
sql_read(statement, principal)       → envelope   (read-only analytical escape hatch)
```

Governance, audit logging, idempotency, and the transaction + outbox frame all
live **inside** the runtime. Surfaces never reimplement them. A REST handler
cannot "forget" to write an audit event, because the REST handler does not
write audit events — the runtime does, on the single code path all three
surfaces share. That shared path is the load-bearing reason isomorphism is
cheap to keep: there is only one place where the real work happens.

The surface-specific part is purely the shape of the request and response on
the wire. REST takes `POST /v1/actions/<name>` with a JSON body `{ "input":
{...} }`. CLI takes `canon <action.name> --flag value`. MCP exposes a single
meta-tool `actions.invoke` with `{ name, args, principal }`. All three decode
their wire format into the same `(action_id, args, principal)` triple and call
`invoke`. See `resources/runtime-contract.md` for the full mapping of each
surface onto each entry point.

## Reaching MCP: per-org endpoints and three credential types

MCP is reached at a **per-org URL**: `https://<host>/mcp/<org-slug>`. One
connection is one org — the org is named in the URL, never chosen by a tool
call — so a request can't silently land on the wrong org. A modern client
(Claude Desktop) points at `https://api.canonify.app/mcp/pax-dev`; a user who
belongs to three orgs adds three separately named connections.

The `/mcp/<org-slug>` door admits **three credential types**, and all three
resolve to the same runtime `principal` before `invoke` ever runs:

- **API key** — the programmatic path. **Agents and service accounts keep using
  API keys**; OAuth does not touch them. This stays the answer for automated
  principals.
- **Central / agent session** — the existing session-bearer path.
- **OAuth access token** — the interactive path. A short-lived Bearer token,
  org-scoped and **RFC 8707 resource-bound** to the URL's org: a token minted
  for org A is rejected on org B's `/mcp/orgB`. It is validated against *that
  org's* DB, and its scopes mirror the user's role in that org (member =
  read+run, admin = manage/authoring, owner = full), so the token grants exactly
  what that person can already do over CLI/REST — no more.

OAuth is **only for interactive clients** that can open a browser (Claude
Desktop first). Don't route a service account through it. And OAuth changes
**nothing about the tool surface**: MCP still exposes the same fixed meta-tools
(`actions.invoke`, `objects.get` / `.list` / `.navigate`, `sql_read`,
`actions.list`, `schema_describe`), and `actions.invoke` still dispatches every
ActionType. OAuth is purely how an interactive client proves *who it is* at the
`/mcp` door; what it can do once inside is the same generated projection of the
Catalog every other surface sees.

The discovery + token endpoints that make the interactive flow zero-config live
alongside `/mcp`:

| Endpoint | RFC | Role |
|---|---|---|
| `GET /.well-known/oauth-protected-resource` | 9728 | Protected Resource Metadata; the `/mcp` 401 carries a `WWW-Authenticate: Bearer resource_metadata="…"` challenge pointing here |
| `GET /.well-known/oauth-authorization-server` | 8414 | Authorization Server metadata — endpoints, grant types, PKCE support |
| `POST /oauth/register` | 7591 | Dynamic Client Registration — the client self-registers, no pre-provisioned `client_id` |
| `GET /oauth/authorize` | 6749 | Browser login + branded consent (client + org + scopes), PKCE; returns an auth code |
| `POST /oauth/token` | 6749 | `authorization_code` and `refresh_token` grants; issues the short-lived access token + refresh token, both resource-bound |
| `POST /oauth/revoke` | 7009 | Revoke a token (incl. refresh) to kill a connection |

For the end-to-end onboarding walk-through, see
`resources/connect-claude-desktop.md`.

## The envelope

Every runtime call — write or read, success or failure — returns the same
envelope shape:

```ts
{
  data,              // the typed object | list | rows (null on error)
  audit_event_id,    // even reads emit a (lighter) audit event
  approval_status,   // 'not_required' | 'pending' | 'approved' | 'rejected'
  knowledge_used,    // [] until the Phase 22 Knowledge primitive populates it
  outcome_signals,   // [] until post-call feedback populates it
  error?,            // { code, message, issues, details? } on failure only
}
```

A successful `customer.create` returns the new row in `data` with
`approval_status: 'not_required'` and the two arrays empty. A failure returns
`data: null` and an `error` with a stable `code` the caller can switch on
(`VALIDATION`, `NOT_FOUND`, `FORBIDDEN`, `INVALID_STATE`, `INTERNAL_ERROR`).
The error envelope is part of the contract, not an exception — REST returns it
with a 4xx status but the *body is still a full envelope*, so a client that
reads the body always sees the same structure regardless of transport.

The deterministic fields of an envelope are identical across surfaces
byte-for-byte. The only fields that legitimately differ between two calls are
the non-deterministic ones — the minted `audit_event_id`, prefixed-ULID row
ids like `cust_…`, and timestamps. See
`resources/envelope-shape-and-error-handling.md` for real success and error
examples taken straight from the live fixtures.

## Why surfaces can't drift

Isomorphism here is not a convention you have to remember. It is enforced by
six mechanisms that make drift structurally hard:

1. **Surfaces are generated**, never hand-written per action. A hand-rolled
   route/command/tool file for an ActionType is a rejected PR.
2. **Each adapter stays trivial** — too small to hide divergent behavior.
3. **A conformance test is the build gate.** One test loops every ActionType ×
   every surface × the action's catalog-declared fixture and asserts the
   normalized envelopes are identical. It fails CI on any mismatch.
4. **Generated ACTION schemas are committed** — `openapi.json` (REST),
   `cli-reference.json` (CLI), `mcp-tools.json` (MCP), generated from the real
   catalog and diffed by CI so an uncommitted delta fails the build. These cover
   the **ActionType surface** (`POST /v1/actions/<name>` and its input schema),
   not the typed read surface: `GET /v1/objects/*`, `navigate`, and `aggregate`
   are projected by the surface adapters at runtime, so they appear in no
   committed schema. Read-surface isomorphism is proven by the conformance
   suite (mechanism 3), not by these files.
5. **The catalog is queryable at runtime** on every surface
   (`/v1/actions`, `canon actions list`, MCP `actions.list`), so cross-surface
   disagreement is detectable live, not just at build time.
6. **AGENTS.md encodes the rule**, so reviewers reject the anti-pattern.

The keystone is mechanism 3. **An ActionType without a fixture does not
ship** — the conformance validator rejects the catalog before any surface is
even invoked. No fixture means the surfaces can't be proven equivalent, so the
action simply doesn't exist.

## Conformance as enforcement

The conformance runner (`tests/conformance/runner.ts`) drives the three
real surface generators as black boxes: it builds a Hono app via
`generateRestApp` and dispatches with `app.request()`, builds the CLI app via
`generateCliApp` and dispatches in-process, and connects an in-memory MCP
`Client` to a `generateMcpServer` instance and calls `actions.invoke`. Each
surface gets its own fresh runtime, so a write on one surface can't bleed into
another's read-set. It then **normalizes** each envelope — masking the
`audit_event_id`, prefixed ULIDs, and timestamps — and compares the results
byte-for-byte. Any divergence is a reported diff and a red build.

This is also the **Phase 17.6 cross-transport conformance gate**: per
AGENTS.md, no phase that adds a new catalog primitive or surface ships without
a passing `tests/scenarios/<name>/session.yaml` proving the primitive works
end-to-end via **both** CLI and REST transports. The conformance test proves
envelope equivalence per action; the scenario gate proves a whole real-world
app works across transports.

## Debugging "works on CLI but not REST"

When a surface seems to misbehave, the divergence is real and locatable —
isomorphism guarantees it shouldn't happen, so finding it usually means
finding a genuine bug. The fastest path:

1. Run the conformance test locally and read the diff — it names the action
   and shows both normalized envelopes side by side.
2. Invoke the action via CLI with `--output json`, grab the
   `audit_event_id` from the envelope, and fetch the full untruncated record
   with `canon audit <id>`.
3. Do the same via REST (`POST /v1/actions/<name>`), grab its
   `audit_event_id`, and fetch it via `GET /v1/audit/<id>`.
4. Diff the two stored envelopes. Because both went through the same runtime,
   a difference points squarely at the surface adapter (wire-parsing or
   formatting), not the business logic.

`resources/debugging-surface-divergence.md` walks this recipe with concrete
commands, plus the checks that rule out the far more common cause: the two
surfaces were called differently (a missing REST `{"input": …}` wrapper, a
different principal, a string-vs-number arg).

## Why this is non-negotiable

Surface isomorphism is one of the architectural commitments in AGENTS.md, not
a nice-to-have. Canonify's reason to exist is letting agents and humans build
and operate apps **declaratively** — one Canon, reached through whichever
transport the client speaks. The moment REST and MCP disagree about what
`deal.win` does, the declarative promise breaks: the Canon is no longer the
single source of truth, and an agent that learned the system over MCP can't
trust the CLI. The conformance gate exists precisely so that promise is
mechanically guaranteed on every push rather than hoped for.
