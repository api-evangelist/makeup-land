---
name: makeup-land-cart-management
description: Read and mutate a customer's persistent makeup.land cart — add a variant, change quantity or tender (ILS vs ℳ-credits), remove a line, clear the cart — idempotently and with stock and wallet gates handled. Bearer + phone required.
api: makeup.land V1 API (https://makeup.land/api/v1)
operations:
  - getCustomer
  - getCart
  - addCartItem
  - patchCartItem
  - deleteCartItem
  - clearCart
  - listProducts
method: generated
generated: '2026-09-19'
source: openapi/makeup-land-openapi.yml, https://makeup.land/llms-full.txt, https://makeup.land/auth.md, conventions/makeup-land-conventions.yml
---

# Manage a customer's cart

## Auth
Every operation here needs BOTH `Authorization: Bearer ml_<hex>` (scope `full`, not `read_only`) AND the customer's E.164 phone. Phone goes in `?phone=%2B972...` for GET/DELETE and in the JSON body for POST/PATCH. Bearer without phone → `400`; phone without bearer → `401`. These writes are NOT available over the MCP server (v1 is read-only; only `get_cart` is exposed).

## Steps
1. **Resolve the customer** (optional but recommended): `GET /api/v1/customers?phone=%2B972501234567` (`getCustomer`) → `credit_balance` (agorot) tells you whether a credits-tender line can be afforded.
2. **Confirm the variant and tender:** `GET /api/v1/products?brand=<slug>` or `?q=...` (`listProducts`) → pick `product_id` + `variant_id`; `credit_price` null means the variant is ILS-only (a `tender: credits` line would 400 `tender_unavailable`).
3. **Read the current cart:** `GET /api/v1/cart?phone=%2B972...` (`getCart`) → `cart_id`, `items[]` with `line_item_id`, `tender`, `quantity`, per-line `reward_projection_cents` and `reward_projection_total_cents`.
4. **Add a line:** `POST /api/v1/cart/items` (`addCartItem`) with header `Idempotency-Key: <uuid4>` and body `{"phone":"+972...","product_id":"...","variant_id":"...","quantity":1,"tender":"ils"}` (optional `gift_personalization`). Response `{ok, cart_id, line_item_id, quantity}`.
5. **Change a line:** `PATCH /api/v1/cart/items/{lineItemId}` (`patchCartItem`) with `Idempotency-Key` and body `{"phone":"+972...","quantity":2}` or `{"phone":"+972...","tender":"credits"}` or a new `variant_id`.
6. **Remove a line:** `DELETE /api/v1/cart/items/{lineItemId}?phone=%2B972...` (`deleteCartItem`) with `Idempotency-Key`.
7. **Clear everything:** `DELETE /api/v1/cart?phone=%2B972...` (`clearCart`) → `{ok, cart_id, cleared_cart_ids[]}`.
8. **Re-read** with `getCart` to confirm the projection before handing off to checkout (checkout itself is not in the V1 API).

## Idempotency and retries
Every write accepts `Idempotency-Key` (UUIDv4, one per logical operation); a retry with the same key within 24 hours replays the original response. Always send it; on a 5xx retry with the SAME key.

## Reversibility
A line you add can be removed with `deleteCartItem` or the whole cart dropped with `clearCart`; a quantity/tender change is undone with another `patchCartItem`. No window is published — the cart is a persistent draft. Nothing here charges money.

## Errors to branch on
- `409 insufficient_stock` → read `available` and `requested`, lower `quantity`.
- `409 insufficient_credits` → read `available_cents` / `requested_cents`, switch the line to `tender: ils` or stop.
- `409 line_collision` → a line for that variant/tender already exists; patch it instead.
- `400 tender_unavailable` / `variant_mismatch` / `invalid_quantity` / `product_unavailable`.
- `404 customer_not_found` (do not retry with the same phone) / `line_item_not_found` (re-read the cart).
- `403 read_only_token` → your token cannot write; request a writable one.
