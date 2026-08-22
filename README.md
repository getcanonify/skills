# Canonify agent skills

The Canonify CLI/REST/MCP manuals, mirrored from the
[`canonify`](https://github.com/pengelbrecht/canonify) monorepo
(`docs/skills/**` is the source of truth). **Do not edit here** — changes are
overwritten on each release. Open issues/PRs against the monorepo.

## Install

```sh
npx skills add getcanonify/skills                  # all skills
npx skills add getcanonify/skills --skill canonify # just the base manual
```

Or as a Claude Code plugin via `.claude-plugin/marketplace.json`.

## Skills

- `canonify`
- `canonify-app-authoring`
- `canonify-esign-sender`
- `canonify-isomorphism`
- `canonify-objects-actions`
- `canonify-scenarios`
- `canonify-viewspecs`

In-product, the same manuals are served by `canon skills` and the public REST
endpoints under `/v1/skill*` / `/v1/skills*`.
