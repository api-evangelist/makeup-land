---
name: makeup-land-gift-card-validate-redeem
description: Check a makeup.land gift card's remaining balance (public, no token) and redeem it against an order with a giftcards-scoped bearer; list a customer's cards.
api: makeup.land V1 API (https://makeup.land/api/v1) and MCP tool validate_gift_card
operations:
  - validateGiftCard
  - redeemGiftCard
  - listGiftCards
method: generated
generated: '2026-09-19'
source: openapi/makeup-land-openapi.yml, https://makeup.land/llms-full.txt, https://makeup.land/pricing.md, mcp/makeup-land-mcp-tools.json
---

# Validate and redeem a gift card

## Auth
- `GET /api/v1/gift-cards/validate?code=...` (`validateGiftCard`) is the API's ONLY fully public route — gated on knowing the code, rate-limited **60/min per IP**. Also the MCP tool `validate_gift_card`.
- `POST /api/v1/gift-cards/redeem` (`redeemGiftCard`) needs a bearer with scope `full` or `giftcards` and not `read_only`.
- `GET /api/v1/gift-cards?phone=%2B972...` (`listGiftCards`) needs bearer + phone.

## Steps
1. **Validate:** `GET /api/v1/gift-cards/validate?code=<full code as printed>` → `{status, balance_cents, initial_amount_cents, currency}`. Never guess codes: missing `code` returns `400` ("code parameter is required", observed); unknown codes return `404`. Back off on `429` (honour `Retry-After`).
2. **Decide the amount:** redemption is in agorot (`amount_cents`, 100 = ₪1) and cannot exceed `balance_cents`.
3. **Redeem:** `POST /api/v1/gift-cards/redeem` with `Idempotency-Key: <uuid4>` and body `{"code":"...","amount_cents":5000,"order_id":"ord_..."}` → `{success, redeemed_cents, remaining_balance_cents}`.
4. **Audit a customer's cards:** `GET /api/v1/gift-cards?phone=...` → `cards[]` with `balance`, `status`, `is_partner_issued`, `order_id` (partner-issued and internal cards are unified).

## Reversibility
There is NO un-redeem operation. pricing.md says a refund process restores a gift card's balance, but that is customer service, not the API. Validate first, redeem once, and always send `Idempotency-Key` so a retry cannot redeem twice.

## Errors
`401 unauthorized`, `403 scope_mismatch` (token lacks `giftcards`/`full`) or `read_only_token`, `400` for a malformed code/amount, `429 rate_limited` on validate. Branch on `error_code`.
