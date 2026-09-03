# Postman Orbit - Codex Plugin

Discover APIs from the [Postman API Network](https://www.postman.com/explore) using **Orbit**, Postman's agent-friendly search API. Built for AI coding agents that need to find the right APIs at the start of a project.

## What is Orbit?

Orbit is a search API designed for AI agent consumption, not human browsing. When you search for a capability like "payment processing," Orbit returns structured results with an `evaluateGuide` field that tells your agent:

- What the endpoint does
- What it is good for ("Use for")
- What it cannot do ("Not supported")

This lets agents make informed decisions about which APIs to integrate without reading documentation pages.

> **Orbit vs `postman:search`:** The `postman:search` skill (from the Postman MCP plugin) searches your personal Postman workspaces and collections. Orbit searches the public Postman API Network and returns results structured for agent decision-making, including the `evaluateGuide` field that explains each API's capabilities and limitations.

## Installation

```bash
codex plugin add Postman-Devrel/orbit-codex-plugin
```

The plugin bundles Orbit's MCP server, so there is nothing else to configure -- no API
key, no `codex mcp add`, no edits to `~/.codex/config.toml`. Installing the plugin
registers the `search` and `integrate` tools, and the skill drives them.

Requires a Codex version that supports plugin-bundled MCP servers over streamable HTTP.
If your Codex only picks up stdio servers, add the server manually instead:

```toml
# ~/.codex/config.toml
[mcp_servers.orbit]
url = "https://mcp.buildwithorbit.ai/mcp"
```

## Usage

Run the `discover` skill with a capability query:

```
/orbit:discover payment processing for subscriptions
```

Search for multiple capabilities at once:

```
/orbit:discover I need APIs for sending transactional emails, geocoding addresses, and generating PDF invoices
```

### Example output

```
### Results for: "send transactional email"

**Send a transactional email** (Brevo)
- Method: `POST`
- URL: `https://api.brevo.com/v3/smtp/email`
- **Evaluate Guide:** Sends a transactional email through Brevo's SMTP API, enabling
  an agent to deliver an email payload to recipients.
  Use for: send transactional messages, deliver notifications, send account emails
  Not supported: inbound email processing, contact management, campaign analytics
```

Results are automatically saved to the `orbit-output/` directory as markdown files for later reference.

Once you have chosen endpoints, the skill can also produce a **task brief** -- the auth
requirements, base URLs, ordered request steps, and gotchas needed to write the
integration.

## Design process

Orbit works best when you use it at the start of a project to build an API blueprint before writing code. Here's the workflow:

1. **Describe what you're building.** Tell your agent the app you want to create, including the key capabilities it needs (payments, auth, email, etc.). You don't need to know which APIs exist yet.

2. **Let the agent query Orbit.** The agent breaks your description into capability queries, hits the Orbit API for each one, and returns candidate endpoints with their evaluateGuide breakdowns.

3. **Read the gaps.** The evaluateGuide's "Not supported" lines are the most valuable part. They tell you what each API can't do, so you can identify missing capabilities before you've written any integration code.

4. **Iterate.** Use those gaps as your next round of queries. "Find me APIs that handle payment refunds" or "I need an auth provider that supports token refresh." Each round narrows the design.

5. **Get the task brief.** Once the endpoint set is settled, the agent sends the selected endpoints plus your task to Orbit's `integrate` tool and gets back a brief covering auth, base URLs, and the request sequence -- the implementation plan, before you write code.

6. **Save the blueprint.** The agent saves results to `orbit-output/` as a structured file you can reference throughout the project. This becomes your API design document, readable by both humans and agents.

The goal is to make API selection decisions intentionally at design time, not discover limitations mid-sprint after you've already integrated half the stack.

## How it works

The plugin is a thin workflow layer over Orbit's MCP server:

| | Provided by |
|---|---|
| `search` / `integrate` tools, request + response schemas | Orbit's MCP server (bundled) |
| Capability decomposition, gap analysis, iteration, saved blueprint | This plugin's skill |

Keeping the API contract on the server side means Orbit can change its parameters
without breaking installed copies of the plugin.

### The underlying API

No authentication is required. The MCP tools map one-to-one onto two REST endpoints on
`https://api.buildwithorbit.ai`:

| MCP tool | REST equivalent |
|---|---|
| `search` | `POST /v1/search` |
| `integrate` | `POST /v1/integrate` |

`search` takes `q` (max 512 chars) plus optional `limit` (default 10, max 25) and
`cursor`, and returns `data[]` entries with `id`, `resourceType`, `name`,
`description`, `method`, `url`, and `evaluateGuide`, alongside `meta` carrying `q`,
`total`, and `nextCursor`. `integrate` takes a `task` and 1-10 `resources` and returns
a `taskBrief`.

If the MCP server is ever unreachable, the skill falls back to these REST endpoints,
documented in [references/orbit-api.md](skills/discover/references/orbit-api.md).

## Links

- [Orbit documentation](https://www.buildwithorbit.ai/)
- [Orbit API reference](https://www.buildwithorbit.ai/api-reference)
- [Postman API Network](https://www.postman.com/explore)
- [Postman](https://www.postman.com)
