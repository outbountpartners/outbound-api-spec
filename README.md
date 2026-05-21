# outbound-api-spec

OpenAPI 3.1 specification for the Outbound Partners Portal API.

This is the **canonical machine-readable contract** for the portal API. It's generated from Zod schemas in the [`outbound-partners-client-dashboard`](https://github.com/outbountpartners/outbound-partners-client-dashboard) edge function and published here for consumption by integrators, the MCP server, ChatGPT custom GPTs, and generated clients.

## Quick links

- **Spec (YAML):** [`openapi.yaml`](./openapi.yaml)
- **Live spec (always reflects deployed code):**
  - JSON: https://sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api/openapi.json
  - YAML: https://sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api/openapi.yaml
- **Manual usage docs:** [outbound-partners-client-dashboard/docs/portal-api-for-n8n.md](https://github.com/outbountpartners/outbound-partners-client-dashboard/blob/staging/docs/portal-api-for-n8n.md)

## What's in the API

Read + write endpoints for:

| Resource | Methods | Scopes |
|---|---|---|
| Meetings | GET, POST, PATCH | `meetings:read`, `meetings:write` |
| Clients | GET, POST, PATCH | `clients:read`, `clients:write` |
| Campaigns | GET, POST, PATCH | `campaigns:read`, `campaigns:write` |
| Users | GET, POST, PATCH | `users:read`, `users:write` (super-admin only) |
| Leaderboard | GET | `leaderboard:read` |
| Commission | GET | `commission:read` |

Plus `/health`, `/whoami`, `/openapi.json`, `/openapi.yaml`.

## Authentication

Every request must include an API key in the `x-api-key` header. Keys are issued by portal super-admins via the `/api` page inside the portal. Each key carries a list of scopes — endpoints reject calls that don't include the required scope with `403 insufficient_scope`.

## Usage

### Browse the spec

```bash
# In your editor
code openapi.yaml

# In Redoc (interactive docs)
npx @redocly/cli preview openapi.yaml

# In Stoplight Studio
open https://elements-demo.stoplight.io/?spec=https://sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api/openapi.json
```

### Generate a TypeScript client

```bash
npx openapi-typescript openapi.yaml -o client.ts
```

### Generate a Python client

```bash
pip install openapi-python-client
openapi-python-client generate --path openapi.yaml
```

### Import into ChatGPT Custom GPT

See the [full ChatGPT setup guide](./manifests/chatgpt-setup.md). Short version:

In the GPT Builder → Configure → Actions → Import from URL, paste:

```
https://sirhkzqpdgarrcyqnjtl.supabase.co/functions/v1/api/openapi.json
```

Set Authentication = API Key, header name `x-api-key`, paste your key.

All write operations carry `x-openai-isConsequential: true`, so ChatGPT prompts the user for confirmation before any POST or PATCH.

### Import into n8n

n8n's HTTP Request node accepts the OpenAPI URL directly. Set Authentication = Header Auth, header name `x-api-key`.

### Use the MCP server

For LLM agent use (Claude Desktop, Claude Code, ChatGPT, n8n MCP node), point at:

```
https://github.com/outbountpartners/outbound-mcp
```

The MCP server reads from this same spec and exposes every endpoint as a typed tool.

## How the spec is updated

The spec is **emitted from Zod schemas** in the portal repo's `supabase/functions/api/openapi.ts`. On every merge to `staging` in the portal repo, the updated YAML is copied here and committed. The runtime `/openapi.json` endpoint always reflects the deployed code, so consumers can either:

- Pull from this repo for stable, reviewed snapshots
- Pull from the live URL for the always-current contract

## Versioning

Current version: `v1`. Breaking changes will be released as `v2` with a 6-month overlap. Additive changes (new fields, new optional params) stay in `v1`.

## Maintainer

Rai — rai@outboundpartners.com
