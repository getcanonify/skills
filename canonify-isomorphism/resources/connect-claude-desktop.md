# Connect Claude Desktop to Canonify

This is the end-to-end path for wiring an **interactive MCP client** (Claude
Desktop first) to one of your Canonify orgs. There is **no API key to paste** —
you add a URL, click through a browser login, and the tools work.

> Building an agent or a service? Don't use this flow — programmatic principals
> keep using **API keys**. OAuth here is for interactive clients that can open a
> browser. See the "Reaching MCP" section of the isomorphism SKILL.

## The short version

1. **Find your org's MCP URL.** In the Canonify dashboard, open **Settings** for
   the org you want to connect. The MCP page shows the URL and the copy-paste
   config. It looks like:

   ```
   https://api.canonify.app/mcp/<your-org-slug>
   ```

   The `<your-org-slug>` part is your org's short, URL-safe identifier (also
   shown in the org's general settings). For the "Pax Dev" org it's `pax-dev`,
   so the URL is `https://api.canonify.app/mcp/pax-dev`.

2. **Add it to Claude Desktop's MCP config.** Give the connection a clear,
   per-org name so you can tell your orgs apart:

   ```json
   {
     "mcpServers": {
       "canonify-pax-dev": {
         "url": "https://api.canonify.app/mcp/pax-dev"
       }
     }
   }
   ```

   The name (`canonify-pax-dev` here) is yours to choose — make it say *which
   org* so a second org's connection is unambiguous.

3. **Log in and consent — that's it.** On first use Claude Desktop registers
   itself automatically and opens a browser. You log into Canonify (if you
   aren't already) and see a **consent screen** naming the client (Claude
   Desktop), the **org** you're connecting, and the **access** it's granting
   (your role in that org — e.g. read + write). Approve it and the browser hands
   the connection back. The tools are now live. **No API key was ever pasted.**

## One org per connection

Each connection is pinned to exactly one org — the org is in the URL and baked
into the token, so a call can never quietly land on the wrong org. To connect a
second org, add **another entry** with its own name and its own `…/mcp/<slug>`
URL:

```json
{
  "mcpServers": {
    "canonify-pax-dev": { "url": "https://api.canonify.app/mcp/pax-dev" },
    "canonify-acme":    { "url": "https://api.canonify.app/mcp/acme" }
  }
}
```

There is nothing to "switch." Each named server is its own org.

## Tokens refresh themselves; revoke anytime

The access token Claude Desktop receives is **short-lived** and **refreshes
automatically** in the background — you won't be asked to log in again for
normal use. If you ever want to cut a connection off, **revoke it** (from
Canonify, or by removing the entry from your config); a revoked token stops
working immediately, refresh included.

## What actually happens under the hood

You don't need any of this to connect, but here's the exact handshake, so a
"why is it asking me to log in?" moment makes sense. It's standard OAuth 2.1
with auto-registration and resource binding:

```
1. Claude Desktop GET  /mcp/pax-dev
      → 401 + WWW-Authenticate: Bearer resource_metadata="…"      (how to auth)
2.                GET  /.well-known/oauth-protected-resource/mcp/pax-dev
      → { authorization_servers, resource }                       (RFC 9728)
3.                GET  /.well-known/oauth-authorization-server
      → endpoints, grant types, PKCE                              (RFC 8414)
4.                POST /oauth/register
      → { client_id }                       (RFC 7591 DCR — zero manual setup)
5.        browser  /oauth/authorize?…&resource=…/mcp/pax-dev&code_challenge=…
      → Canonify login + CONSENT (org = Pax Dev, scope = your role)
      → redirect back with ?code=…
6.                POST /oauth/token   (code + PKCE verifier + resource)
      → { access_token (short), refresh_token, token_type: Bearer, expires_in }
7.                GET/POST /mcp/pax-dev   with Bearer access_token
      → validated: token's resource == /mcp/pax-dev, principal = you, org =
        pax-dev, roles ← scope. Tools work. On expiry, step 6 with the refresh
        token mints a new access token — no re-consent.
```

The one invariant worth knowing: the token is **bound to this org's URL**
(RFC 8707 Resource Indicators). A token issued for `…/mcp/pax-dev` is **rejected**
if it's ever presented to `…/mcp/acme`. The org in the URL is a routing
address, never a stand-in for a credential — which is exactly why one
connection can only ever act as its one org.
