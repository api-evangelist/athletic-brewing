---
name: athletic-brewing-catalog-discovery
description: >-
  Find non-alcoholic beers and merchandise in the Athletic Brewing store and resolve them
  to purchasable variant IDs, using the store's live UCP/MCP endpoint.
api: Athletic Brewing UCP Commerce (MCP)
endpoint: https://athleticbrewing.com/api/ucp/mcp
generated: '2026-08-06'
method: generated
source: mcp/athletic-brewing-tools-list.json
operations:
  - search_catalog
  - lookup_catalog
  - get_product
---

# Athletic Brewing — catalog discovery

Read-only. No buyer credentials are needed for any step here. Every tool name and every
parameter below is taken from the `tools/list` response the endpoint actually returned
(`mcp/athletic-brewing-tools-list.json`) — nothing is invented.

## Before you call anything

Every `tools/call` must carry agent identity:

```json
"meta": { "ucp-agent": { "profile": "https://your-agent.example/ucp-profile" } }
```

Omit it and the endpoint returns **HTTP 422**, JSON-RPC `-32001` `UCP discovery failed`,
`data.code = invalid_profile_url`. That failure happens before method dispatch, so it will
look identical whether your method name was wrong or your profile was missing.

`initialize` and `tools/list` are anonymous — use them to confirm the server is up and the
tool set has not changed (`serverInfo.name` is `universal-commerce`).

## Steps

1. **Search.** Call `search_catalog` with `catalog.query` in natural language.
   - Always set `catalog.context.address_country` (ISO 3166-1 alpha-2) and
     `catalog.context.currency` (ISO 4217). Pricing and availability are wrong without them.
   - Narrow with `catalog.filters.categories` (OR logic), `catalog.filters.price.min` /
     `.max` — **integers in minor currency units**, so $12.99 is `1299` — and
     `catalog.filters.available` (defaults `true`, sale-ready items only).
   - Page with `catalog.pagination.limit` (default `10`, minimum `1`) and
     `catalog.pagination.cursor`. No maximum limit is published; do not assume one.
2. **Resolve in bulk.** When you already hold several product or variant identifiers, call
   `lookup_catalog` once rather than `get_product` N times.
3. **Get the detail you will buy from.** Call `get_product` for the item the buyer picked.
   This is the call that returns the full variant set and exact pricing — the variant `id`
   it returns is what `create_cart` needs as `cart.line_items[].item.id`.

## Rules that are the store's, not ours

- The MCP endpoint is **rate-limited per IP**. Back off on `429`. No numeric limit is published.
- Do not scrape the storefront HTML to do this. The store's own `robots.txt` and `llms.txt`
  direct agents to the UCP/MCP endpoints instead.
- If you only need to read, stay here. Do not touch cart or checkout tools.

## Alternate read paths the store publishes

For plain catalog reads with no MCP client at all, the store documents these in `/llms.txt`:
`GET /products/{handle}.json`, `GET /collections/{handle}/products.json`,
`GET /search?q={query}&type=product`, `GET /sitemap.xml`. All anonymous.

## Next

To buy, hand off to `athletic-brewing-guided-checkout`.
