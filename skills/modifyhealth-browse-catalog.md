---
name: Browse the ModifyHealth meal catalog
description: Search and inspect ModifyHealth's dietitian-crafted meal programs over its public UCP commerce MCP endpoint, without authentication.
api: mcp/modifyhealth-mcp.yml
endpoint: https://modifyhealth.com/api/ucp/mcp
operations: [search_catalog, lookup_catalog, get_product]
generated: '2026-08-26'
method: generated
source: mcp/modifyhealth-mcp-tools.json (live tools/list, HTTP 200)
---

# Browse the ModifyHealth meal catalog

ModifyHealth sells prepared, dietitian-crafted meal programs — Low FODMAP (Monash
certified), Mediterranean, gluten-free, heart-friendly, diabetes-friendly, carb-conscious
and GLP-1 support. The catalog is readable by an agent with no credential.

## Connect

POST JSON-RPC 2.0 to `https://modifyhealth.com/api/ucp/mcp` with
`Content-Type: application/json` and `Accept: application/json, text/event-stream`.

Every tool call must carry `meta.ucp-agent.profile` — a resolvable URI identifying your
agent. Omitting it returns JSON-RPC error `-32001` / `invalid_profile_url`.

## Steps

1. **Confirm capability.** `GET https://modifyhealth.com/.well-known/ucp` and check that
   `ucp.services["dev.ucp.shopping"]` lists a `transport: mcp` endpoint, and that
   `ucp.version` is a version you support (currently `2026-04-08`; `2026-01-23` is also
   served).
2. **Search.** Call `search_catalog` with the buyer's intent. Pass
   `context.address_country` and `context.currency` so pricing and availability come back
   correct for the buyer.
3. **Resolve detail.** Call `get_product` with an identifier from the search result to get
   full product detail, or `lookup_catalog` to resolve several products or variants in one
   round trip.

## Rules

- **Prices are integers in ISO 4217 minor units.** `{"amount": 5250, "currency": "USD"}`
  is $52.50. Divide by 100 for USD/EUR before quoting a buyer. Never show the raw integer.
- **Line items reference VARIANT ids, not product ids.** `get_product` is where you find
  the variant id you will later add to a cart.
- **Back off on 429.** The endpoint is rate limited per IP and returns no
  `RateLimit-*` headers, so you get no advance warning of your budget.
- **Errors arrive with HTTP 200.** Inspect the JSON-RPC `error` member; do not trust the
  status code. An `error.data.continue_url` is a browser URL you can hand to the buyer.

See `conventions/modifyhealth-conventions.yml` and
`errors/modifyhealth-problem-types.yml`.
