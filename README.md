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
### Results for: "payment processing"

**Stripe - Create Subscription**
- Method: `POST`
- URL: `https://api.stripe.com/v1/subscriptions`
- Description: Creates a new subscription on an existing customer.
- **Evaluate Guide:** Use for: recurring billing, subscription lifecycle management,
  plan upgrades/downgrades. Not supported: one-time payments (use Payment Intents),
  physical goods shipping, tax calculation (use Stripe Tax).
```

Results are automatically saved to the `orbit-output/` directory as markdown files for later reference.

## The Orbit API

The plugin calls a single endpoint:

```
POST https://fabric-gateway.postmanlabs.com/api/search
Content-Type: application/json

{"q": "your search query"}
```

No authentication required. The response includes:

- `data[]` - Array of API endpoints with `id`, `name`, `description`, `method`, `url`, and `evaluateGuide`
- `meta` - Search metadata with `q`, `total`, and `nextCursor`

## Links

- [Postman API Network](https://www.postman.com/explore)
- [Postman](https://www.postman.com)
