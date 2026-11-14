---
name: canonify-isomorphism
description: Why REST, CLI, and MCP are three isomorphic projections of one Canon catalog — the runtime contract they all call, the envelope they all return, why they can't drift, and how to debug 'works on CLI but not REST'. Read SKILL.md first, then deep-dive on the runtime contract or connecting an MCP client.
version: 2.0.0
---

# Canonify REST/CLI/MCP Isomorphism

One Canon — ObjectTypes, ActionTypes, views, Apps — is reachable over REST,
the `canon` CLI and MCP. **The same operation with the same input produces the
same result on every transport.** Pick the surface that fits your client; every
surface supports every feature by construction.

| Page | What |
|---|---|
| [resources/runtime-contract.md](resources/runtime-contract.md) | The five runtime entry points, how each surface maps onto them, the REST body and read filters, the NDJSON progress stream, principals |
| [resources/connect-claude-desktop.md](resources/connect-claude-desktop.md) | MCP endpoints and credentials; connecting an interactive client with per-org OAuth |

## Three surfaces, one catalog

REST, CLI and MCP are **generated projections** of the catalog, not three
codebases that agree. No ActionType has a hand-written route, command or tool:
declare one (or let auto-CRUD generate `create` / `read` / `update` / `delete`
/ `list` from a new table) and it is live on all three at once, with no
restart. Each surface's own code is a thin adapter — parse the principal, parse
the args against the action's schema, call the runtime, format the envelope.
So "is X on MCP?" is the wrong question: if X is in the catalog it is on every
surface; if not, on none.

Governance, audit, idempotency and the transaction + outbox frame live inside
the one runtime all three call, never in an adapter. A REST handler cannot
forget to audit, because adapters don't audit; the runtime does.

## One envelope

Every call on every surface returns the same envelope (fields: the base
skill, [../canonify/SKILL.md](../canonify/SKILL.md); errors:
[../canonify/resources/error-codes.md](../canonify/resources/error-codes.md)).
For the same call, the deterministic fields are byte-identical across
surfaces; only the minted `audit_event_id`, minted row ids and timestamps
differ. A REST failure arrives with a 4xx status and a **full envelope body** —
switch on `error.code`, not on the transport's status.

## Why surfaces can't drift

1. **Generated, never hand-written** per action.
2. **Adapters stay trivial** — too small to hide divergent behaviour.
3. **A conformance gate** invokes every ActionType on all three surfaces with
   its declared fixture, masks the minted ids, audit ids and timestamps, and
   requires the envelopes to match byte for byte. **An ActionType without a
   fixture does not ship.**
4. **Committed wire schemas** — `openapi.json`, `cli-reference.json`,
   `mcp-tools.json` — are generated from the catalog and diffed on every build.
   They cover the ActionType surface (`POST /v1/actions/<name>`), not the typed
   reads (`GET /v1/objects/*`, navigate, aggregate), which the adapters project
   at runtime; the conformance gate covers those.
5. **Discovery is live** on every surface (`GET /v1/actions`,
   `canon actions list`, MCP `actions.list`), so disagreement is detectable at
   runtime.

## Debugging "works on CLI but not REST"

Most reported divergences are calls that differ. Rule these out first:

- **REST wraps its args**: `POST /v1/actions/<name>` takes `{"input": {…}}`
  (the body rule is in [runtime-contract.md](resources/runtime-contract.md)).
- **The principal differs**: the CLI runs as your saved session, REST as the
  bearer's principal — different governance outcomes are correct.
- **Coercion differs**: CLI flags arrive as strings; `--count 3` and
  `{"count": "3"}` may not agree.

If inputs and principal really are identical, diff what the runtime stored:

```bash
canon customer.create --name Alice --email alice@example.com --output json   # note audit_event_id
canon audit <audit_event_id> --output json        # sugar over objects get _audit_event <id>; `result` = the envelope data

curl -X POST $BASE/v1/actions/customer.create -H "authorization: Bearer $KEY" \
  -H 'content-type: application/json' -d '{"input":{"name":"Alice","email":"alice@example.com"}}'
curl $BASE/v1/objects/_audit_event/<audit_event_id> -H "authorization: Bearer $KEY"
```

Mask the fields that legitimately differ (`audit_event_id`, minted ids,
timestamps). Whatever else differs sits in a surface adapter's parse/format
seam, not in your action — report it with both audit ids and both envelopes
(`canon report`).
