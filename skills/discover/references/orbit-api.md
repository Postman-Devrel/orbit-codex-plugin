# Orbit API Reference — REST fallback

Docs: https://www.buildwithorbit.ai/api-reference
OpenAPI: https://www.buildwithorbit.ai/openapi.json

> **Prefer the MCP tools.** This plugin bundles Orbit's MCP server, so the `search`
> and `integrate` tools are normally available and carry live, self-describing
> schemas. Use the REST calls below only when those tools are unavailable. See
> [MCP tools](#mcp-tools) at the end for the mapping.

**Base URL:** `https://api.buildwithorbit.ai`

No authentication is required. Only `Content-Type: application/json` is needed.

There are two endpoints: `/v1/search` finds candidate endpoints, `/v1/integrate` turns
the ones you pick into a task brief.

---

## POST /v1/search

Describe your goal in `q`. Returns matching public endpoints, each with an
`evaluateGuide` explaining what it does, when to use it, and its limitations.

### Query parameters

| Parameter | Type | Default | Notes |
|-----------|------|---------|-------|
| `limit` | integer | 10 | Results per page. Min 1, max 25. |
| `cursor` | string | — | Pass `meta.nextCursor` from the previous response. Omit for the first page. |

### Request body

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `q` | string | Yes | Natural language query, keywords, an API name, or a question. 1–512 characters. |

`q` is the only accepted body field — unknown fields return `400`.

```json
{ "q": "Add tracking details for an existing paypal order" }
```

### Example curl

```bash
curl -s -X POST 'https://api.buildwithorbit.ai/v1/search?limit=10' \
  -H 'Content-Type: application/json' \
  -d '{"q": "payment processing"}' | jq .
```

### Response (200)

```json
{
  "data": [
    {
      "id": "urn:orbit:endpoint:v1:...:brevo:send-a-transactional-ema",
      "resourceType": "endpoint",
      "name": "Send a transactional email",
      "description": "string",
      "method": "POST",
      "url": "https://api.brevo.com/v3/smtp/email",
      "evaluateGuide": "string",
      "provider": "Brevo",
      "product": "Brevo"
    }
  ],
  "meta": {
    "q": "send transactional email",
    "total": 2,
    "nextCursor": "eyJmcm9tIjoyfQ=="
  }
}
```

| Field | Description |
|-------|-------------|
| `id` | Opaque identifier of the form `urn:orbit:endpoint:v1:...`. Pass it back verbatim to `/v1/integrate` — never parse or construct it. |
| `resourceType` | Kind of entity, e.g. `endpoint`. Pass it to `/v1/integrate` as the resource's `type`. |
| `name` | Human-readable name of the endpoint |
| `description` | What the endpoint does (may be empty) |
| `method` | HTTP method used to call the endpoint |
| `url` | URL the endpoint is called at |
| `evaluateGuide` | Three-part evaluation: brief summary, recommended use cases, unsupported use cases/limitations |
| `provider` / `product` | Owning provider and product (returned by the live API; not in the published OpenAPI spec, so treat as optional) |

`meta` carries `q`, `total`, and `nextCursor`. **`nextCursor` is absent on the last
page** — check for its presence rather than comparing to `null`.

### Pagination

Pass `nextCursor` as the `cursor` **query parameter** (not a body field):

```bash
curl -s -X POST 'https://api.buildwithorbit.ai/v1/search?cursor=eyJmcm9tIjoyfQ==' \
  -H 'Content-Type: application/json' \
  -d '{"q": "payment processing"}'
```

Pagination stops at 40 results total; a cursor past 40 is rejected.

### Errors

`400` invalid input · `429` rate limited · `500` server error.
Error bodies are RFC 9457 problem details: `type`, `title`, `status`, `detail`, `instance`.

---

## POST /v1/integrate

After selecting endpoints from `/v1/search`, send them here along with the task you
want to accomplish. Returns a **task brief** with the information and steps needed to
call those endpoints.

Takes no query parameters.

### Request body

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `task` | string | Yes | What you want to accomplish. 1–512 characters, must contain non-whitespace. |
| `resources` | array | Yes | Endpoints to integrate, 1–10 items. Each item needs `id` and `type`. The MCP tool schema enforces a max of 10; the published OpenAPI spec omits the limit, so assume it applies to REST too and split larger sets across calls. |
| `resources[].id` | string | Yes | The `id` from a `/v1/search` result, verbatim. |
| `resources[].type` | string | Yes | The result's `resourceType`. Currently only `endpoint`. |

```json
{
  "task": "Build an app to post current weather to Slack",
  "resources": [
    { "id": "urn:orbit:endpoint:v1:...:weatherapi-com:current-weather-json", "type": "endpoint" },
    { "id": "urn:orbit:endpoint:v1:...:slack:send-message-to-slack", "type": "endpoint" }
  ]
}
```

### Example curl

```bash
curl -s -X POST https://api.buildwithorbit.ai/v1/integrate \
  -H 'Content-Type: application/json' \
  -d '{
    "task": "Send a transactional email when a user signs up",
    "resources": [
      { "id": "urn:orbit:endpoint:v1:...:sendmux:send-a-single-email", "type": "endpoint" }
    ]
  }' | jq -r '.data[0].taskBrief'
```

### Response (200)

```json
{
  "data": [
    { "taskBrief": "string" }
  ]
}
```

The `taskBrief` is a multi-line document covering FIT, AUTH (including which
credentials you must supply), BASE URL, STEPS with parameters and expected responses,
dependencies between steps, and important considerations. It is built from the
selected endpoints' schemas plus shared variables, auth settings, and descriptions
defined by their parent APIs.

### Errors

`400` invalid input · `404` none of the `id`s could be resolved · `429` rate limited ·
`500` server error.

---

## Idempotency

Both endpoints are read-only and never create or mutate data, so retries are safe and
no idempotency key is needed. Search results may change as the public catalog changes;
task brief wording may vary between otherwise identical calls.

## The evaluateGuide field

`evaluateGuide` is what makes Orbit results agent-friendly. It is a newline-separated
string in three parts, so an agent can decide whether an API fits without
trial-and-error:

- **Summary** — a concise description of the endpoint's purpose
- **`Use for:`** — specific scenarios where this endpoint is the right choice
- **`Not supported:`** — capabilities this endpoint does not cover, preventing wasted
  integration effort

Example:

```
Sends a transactional email through Brevo's SMTP API, enabling an agent to deliver an email payload to recipients.
Use for: send transactional messages, deliver notifications, send account emails
Not supported: inbound email processing, contact management, campaign analytics
```

---

## MCP tools

The plugin bundles Orbit's MCP server (`https://mcp.buildwithorbit.ai/mcp`, streamable
HTTP transport, no auth) via `.mcp.json`, referenced from `.codex-plugin/plugin.json`
as `"mcpServers": "./.mcp.json"`. It exposes two tools that map one-to-one onto the
REST endpoints and return identical payloads:

| MCP tool | REST equivalent |
|----------|-----------------|
| `search` | `POST /v1/search` |
| `integrate` | `POST /v1/integrate` |

Differences from REST:

- `limit` and `cursor` are ordinary tool arguments, not query parameters.
- Both tools accept an optional `clientName` string for anonymous usage analytics.
  Pass `"codex/orbit-plugin"`.
- `integrate` declares `resources` as 1–10 items in its schema.

The tool schemas are the authoritative contract — they are fetched live from the
server, so they stay correct even when this file drifts.

### Query guidance (from the tool description)

- Use focused keyword queries including the product or provider name plus the endpoint
  detail — e.g. `"PayPal create invoice"`.
- Natural language works too — e.g. `"PayPal API to create an invoice"`.
- Avoid jumbled queries cramming unrelated keywords together — e.g.
  `"paypal invoice payment delivery payments ordering"`.
- Avoid `OR`-separated queries — e.g. `"paypal invoice OR paypal create invoice"`.
- To explore multiple intents, make a separate call per intent.
