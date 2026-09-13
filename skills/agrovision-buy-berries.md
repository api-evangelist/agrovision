---
name: agrovision-buy-berries
description: >-
  Search the Fruitist (Agrovision) direct-to-consumer berry store, build a cart, and take a
  checkout right up to — but never through — the buyer approval step, using the store's live
  Universal Commerce Protocol MCP endpoint.
api: agrovision-ucp-commerce
endpoint: https://shop.fruitist.com/api/ucp/mcp
protocol: UCP 2026-08-25 over MCP (JSON-RPC 2.0)
auth: none for catalog, cart and checkout construction
generated: '2026-09-13'
method: generated
source: >-
  Grounded in the live tools/list response saved at mcp/agrovision-ucp-mcp-tools.json
  (HTTP 200, 2026-09-13) — every tool name below was returned by the server.
operations:
- search_catalog
- lookup_catalog
- get_product
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

# Buying Fruitist berries as an agent

Fruitist (formerly Agrovision) sells jumbo blueberries, raspberries, blackberries and cherries
direct to consumers from a Shopify store that speaks the Universal Commerce Protocol. There is
no other API. Everything below runs against one endpoint:

```
POST https://shop.fruitist.com/api/ucp/mcp
Content-Type: application/json
Accept: application/json, text/event-stream
```

## Before the first call

Every tool in this store marks `meta` **required**, and requires `meta.ucp-agent.profile` — a URI
identifying your agent — inside it. A call without it will be rejected. Pass the buyer's
`context.address_country` and `context.currency` too, or prices and availability will be wrong.

Confirm the store's capabilities first with `GET https://shop.fruitist.com/.well-known/ucp`; it
names the protocol version in force and the capability set (`dev.ucp.shopping.cart`,
`dev.ucp.shopping.checkout`, `dev.ucp.shopping.fulfillment` and so on).

## 1. Find the product

- `search_catalog` — free-text search over this store's products.
- `lookup_catalog` — resolve several known identifiers at once.
- `get_product` — full detail for one product or variant.

The catalog is small (snack packs, jumbo blueberries, cherry boxes and similar), so search
rarely needs paging.

## 2. Build the cart

- `create_cart` returns a server-assigned cart id (`gid://shopify/...`). Keep it.
- `update_cart` changes lines on that id. Repeating an update converges on the same state —
  useful, but this is **not** idempotency: there is no `Idempotency-Key` header on this surface,
  so never rely on retry-safety for a create call. If a `create_*` call times out, read back
  before creating again.
- `get_cart` to re-read, `cancel_cart` to abandon it.

## 3. Build the checkout

- `create_checkout` returns totals, taxes and any applicable discounts.
- `update_checkout` sets the shipping address and delivery method. Fulfillment on this store is
  **shipping only, single destination** — the merchant profile declares
  `method_combinations: [["shipping"]]` and `multi_destination: []`. Do not attempt a split
  shipment or a pickup method.
- `get_checkout` to re-read state at any point.

## 4. Stop at payment

`complete_checkout` is the one irreversible step. The store's own agent instructions are
explicit: *"Agents must not complete payment without explicit buyer consent."* Get
contemporaneous buyer approval, or hand the purchase to the buyer's own Shop Pay flow instead.

## 5. Backing out

- Before completion: `cancel_checkout` and `cancel_cart` both exist and work. **No window is
  published** for either — do not promise the buyer one.
- After completion: there is no reversal tool on this endpoint. `get_order` is read-only.
  Refunds run through the store's human refund policy at
  <https://shop.fruitist.com/policies/refund-policy>, not through the agent surface. Say so
  plainly before the buyer approves payment.

## Reading money correctly

Prices come back as integers in ISO 4217 **minor** units paired with a currency code:
`{"amount": 600, "currency": "USD"}` is $6.00. Divide by 100 for two-decimal currencies before
quoting a price to a human; zero-decimal currencies such as JPY are already whole units.
Misreading this is the most likely way to quote a buyer a price 100× wrong.

## Errors and limits

Transport errors are JSON-RPC 2.0 error objects; business failures come back inside the
`complete_checkout` result. The endpoint is rate-limited **per IP** with no published ceiling —
back off on `429`. No `RateLimit-*` or `Retry-After` headers were returned on a successful call,
so treat the 429 itself as the only signal.
