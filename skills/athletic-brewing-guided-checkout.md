---
name: athletic-brewing-guided-checkout
description: >-
  Build a cart and drive a checkout to completion on the Athletic Brewing store over
  UCP/MCP, with the buyer approving payment and an idempotency key on the completing call.
api: Athletic Brewing UCP Commerce (MCP)
endpoint: https://athleticbrewing.com/api/ucp/mcp
generated: '2026-08-06'
method: generated
source: mcp/athletic-brewing-tools-list.json
operations:
  - create_cart
  - get_cart
  - update_cart
  - cancel_cart
  - create_checkout
  - get_checkout
  - update_checkout
  - complete_checkout
  - cancel_checkout
  - get_order
---

# Athletic Brewing — guided checkout

This flow moves money. Read the approval rule first.

> **Checkout requires human approval.** The store states, in its own `/llms.txt`,
> `/agents.md` and `/robots.txt`: agents must not complete checkout, payment, or order
> placement without an explicit, contemporaneous human approval step — no scripted form
> fills, no end-to-end automation that finalizes payment. If you cannot get that approval at
> the moment of payment, do not call `complete_checkout`.

Every tool name and parameter below comes from the endpoint's own `tools/list` response
(`mcp/athletic-brewing-tools-list.json`).

## Preconditions

- `meta.ucp-agent.profile` on **every** call — a resolvable UCP agent profile URI. Missing
  or unresolvable returns HTTP 422 / JSON-RPC `-32001`.
- Variant IDs resolved via `athletic-brewing-catalog-discovery`.
- Buyer-scoped account operations need Shopify customer-account OpenID Connect (PKCE S256,
  scopes `openid`, `email`, `customer-account-api:full`, `customer-account-mcp-api:full`) —
  see `authentication/athletic-brewing-authentication.yml`.

## Steps

1. **`create_cart`** with `cart.line_items[]`, each `{ item: { id: <variant id> }, quantity }`.
   Optionally set `cart.buyer.email` / `cart.buyer.phone_number` and `cart.context`
   (`address_country`, `address_region`, `postal_code`, `language`, `intent`) as provisional
   hints — the schema notes higher-resolution data later supersedes them.
   **There is no idempotency key on this tool.** A blind retry can create a second cart;
   read back with `get_cart` before retrying.
2. **`get_cart` / `update_cart`** to adjust quantities or line items. `update_cart` needs
   both `id` and `cart`. `cancel_cart` abandons it.
3. **`create_checkout`** to turn the cart into a checkout. The response carries line items,
   totals, discounts and taxes. Same caveat as step 1 — no idempotency key here either.
4. **`update_checkout`** to set the shipping address and fulfillment method. This store
   declares `dev.ucp.shopping.fulfillment` with `allows_multi_destination.shipping = false`
   and a single allowed method combination (`["shipping"]`) — one destination, shipping
   only. Do not attempt split shipments or pickup.
5. **Show the buyer the real totals and get explicit approval.** Present the amounts
   returned by `get_checkout`, not amounts you computed. Money is in minor currency units.
6. **`complete_checkout`** — and only now:
   - `meta.idempotency-key` is **required** on this tool. It is the only one of the thirteen
     that requires it. Generate a stable key per approved purchase attempt and reuse that
     same key on any retry.
   - No replay-retention window is published, so do not assume a stale key is still honored.
   - Payment runs through the handlers the store declares: `dev.shopify.shop_pay`,
     `dev.shopify.card` (visa, master, american_express, discover, diners_club) and Google
     Pay. Your agent never handles card data.
7. **`get_order`** with the returned order ID to confirm and to track.
8. **`cancel_checkout`** if the buyer backs out before completion.

## Failure handling

- `HTTP 422` + `-32001` `invalid_profile_url` — your agent profile is missing or
  unresolvable. Fix the profile; retrying the same call will not help.
- `HTTP 429` — per-IP rate limit. Back off. No published limit or `Retry-After`.
- Errors are JSON-RPC objects (`error.code`, `error.message`,
  `error.data.{code,content,continue_url}`), **not** RFC 9457 problem details. When
  `continue_url` is present, that is the URL to hand a human.
- Correlate support requests with the `x-request-id` response header.

## If you are a personal shopping assistant

The store explicitly recommends installing `https://shop.app/SKILL.md` and routing the
purchase through Shop Pay instead of driving these tools yourself, because that path carries
the buyer-approval invariant for you. Prefer it when your user permits installation.
