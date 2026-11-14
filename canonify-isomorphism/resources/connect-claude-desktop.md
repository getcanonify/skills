# MCP access and connecting Claude Desktop

## Endpoints and credentials

MCP lives at a **per-org URL**, `https://<host>/mcp/<org-slug>`: one connection
is one org, named in the URL, never chosen by a tool call. A bare
`https://<host>/mcp` also works for API-key and agent-session callers (the org
comes from the credential); OAuth is not admitted there, because its resource
binding needs the org in the URL.

`/mcp/<org-slug>` admits three credentials, all resolved to the same principal
before the runtime runs:

- **API key** — the path for agents and service accounts; OAuth never touches
  them. Narrow a key with `canon key create --type agent --name <n> --role
  member|admin`.
- **Central / agent session** — the session-bearer path.
- **OAuth access token** — only for interactive clients that can open a
  browser. Short-lived, org-scoped and resource-bound (RFC 8707): a token for
  org A is rejected on `/mcp/orgB`. Its scopes mirror the user's role in that
  org (member = read + run, admin = manage/authoring, owner = full).

The tool surface is the same for every credential: `actions.invoke`,
`actions.list`, `objects.get` / `.list` / `.list_types` / `.navigate` /
`.search` / `.aggregate` / `.attachments`, `sql_read`, `schema_describe`.

| Endpoint | RFC | Role |
|---|---|---|
| `GET /.well-known/oauth-protected-resource` | 9728 | resource metadata; a `/mcp` 401 carries `WWW-Authenticate: Bearer resource_metadata="…"` pointing here |
| `GET /.well-known/oauth-authorization-server` | 8414 | endpoints, grant types, PKCE |
| `POST /oauth/register` | 7591 | dynamic client registration — no pre-provisioned `client_id` |
| `GET /oauth/authorize` | 6749 | browser login + consent (client, org, scopes), PKCE |
| `POST /oauth/token` | 6749 | `authorization_code` and `refresh_token` grants; both tokens resource-bound |
| `POST /oauth/revoke` | 7009 | revoke a token, refresh included |

## Connect Claude Desktop

No API key is pasted.

1. In the Canonify dashboard, open the org's **Settings → MCP** for the URL,
   e.g. `https://api.canonify.app/mcp/pax-dev`.
2. Add it to Claude Desktop's config under a name that says which org:

   ```json
   {
     "mcpServers": {
       "canonify-pax-dev": { "url": "https://api.canonify.app/mcp/pax-dev" },
       "canonify-acme": { "url": "https://api.canonify.app/mcp/acme" }
     }
   }
   ```

   Each entry is its own org; there is nothing to switch.
3. On first use the client registers itself and opens a browser: log in,
   approve the consent screen (client, org, the access your role gives), done.

The access token refreshes in the background without re-consent. Revoke a
connection from Canonify (or delete the entry); a revoked token stops working
at once, refresh included.

Under the hood: `GET /mcp/<slug>` → 401 with the metadata challenge →
resource metadata → authorization-server metadata → `POST /oauth/register` →
browser `/oauth/authorize` (with `resource` and a PKCE challenge) →
`POST /oauth/token` → `/mcp/<slug>` with the Bearer token, validated against
that org. On expiry the refresh token mints a new access token.
