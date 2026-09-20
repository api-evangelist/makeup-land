---
name: makeup-land-partner-registration
description: Register or look up a makeup.land customer from a partner system with a register-scoped bearer, set tags, and follow the resulting webhook / WhatsApp delivery status and M Club opportunities.
api: makeup.land V1 API (https://makeup.land/api/v1)
operations:
  - registerCustomer
  - getRegistration
  - listRegistrations
  - listCustomerOpportunities
  - patchCustomerTags
method: generated
generated: '2026-09-19'
source: openapi/makeup-land-openapi.yml, https://makeup.land/llms-full.txt, https://makeup.land/pricing.md ("Earn lifecycle"), asyncapi/makeup-land-webhooks.yml
---

# Register a customer as a partner

## Auth
Bearer with scope `register` (or `full`), not `read_only`, and — for `registerCustomer` — a `registration_source` configured on the token by makeup.land (otherwise `403`). Registrations you list or read are limited to your token's `registration_source`.

## Steps
1. **Register (or upsert):** `POST /api/v1/register` (`registerCustomer`) with `Idempotency-Key: <uuid4>` and body `{"phone":"+972...","name":"...","email":"...","tags":["first-touch"]}` (optional `tag`, `team: "yes"`, `profile_pic_url` — HTTPS on an allow-listed WhatsApp/Facebook CDN host only —, `referred_by_code`, `referral_source`). `201` = created, `200` = existing customer updated. Response: `{status:"ok", customer, created, webhook:{status,status_code,attempts,error}, whatsapp:{status,template,error}, tags_added, tags_already_present}`.
2. **Know what fires:** the partner webhook and the WhatsApp template fire ONLY when `created` is true; on first insert the M Club static opportunities are materialised. The ℳ10 welcome bonus is minted only on phone-verified paths (this endpoint after WhatsApp delivery, OTP login, admin create) — never via `POST /api/v1/customers`.
3. **Follow up on delivery:** `GET /api/v1/registrations/{id}` (`getRegistration`) → `webhook {status, url, status_code, attempts}` and `whatsapp {status, template}`. Or page all with `GET /api/v1/registrations?status=&tag=&from=&to=&limit=&page=` (`listRegistrations`) → `total`, `page`, `has_more`.
4. **Adjust tags later:** `PATCH /api/v1/customers/{id}/tags` (`patchCustomerTags`) with `{"add":[...],"remove":[...]}` or `{"replace":[...]}`; the response echoes `tags_added` / `tags_removed`, so the inverse call is always known.
5. **Surface offers:** `GET /api/v1/customers/{id}/opportunities` (`listCustomerOpportunities`, 60/min per token) → `opportunities[]` (Hebrew title/description, bonus, CTA) and the customer's `referral {code, share_url_wa}`.

## Idempotency and rate limits
Send `Idempotency-Key` on `registerCustomer` and `patchCustomerTags` (24h replay). `registerCustomer` declares `429` (limit unpublished) and `listCustomerOpportunities` is 60/min per token — honour `Retry-After`.

## Reversibility
There is no delete-customer or cancel-registration operation. Tag changes are reversible with `patchCustomerTags`; the registration itself is not. Confirm the phone before you call.

## Errors
`409 phone_conflict`, `403` (`scope_mismatch`, `read_only_token`, or no `registration_source`), `400 invalid_phone` (must be E.164), `404 registration_not_found`.
