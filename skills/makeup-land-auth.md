# auth.md

> Agentic authentication manifest for the makeup.land V1 API, per the
> WorkOS `auth.md` skill format.

This document tells AI agents how to obtain and use credentials for the
makeup.land V1 API (`https://makeup.land/api/v1/`). It complements the OpenAPI
specification at [https://makeup.land/openapi.json](https://makeup.land/openapi.json) and the
RFC 9728 / RFC 8414 metadata at
[https://makeup.land/.well-known/oauth-protected-resource](https://makeup.land/.well-known/oauth-protected-resource)
and
[https://makeup.land/.well-known/oauth-authorization-server](https://makeup.land/.well-known/oauth-authorization-server).

**Honest statement of support.** Today makeup.land issues bearer tokens
through a human-mediated path. There is no programmatic
`/agent/auth` registration endpoint, no OTP claim ceremony, and no
anonymous flow. Agents that need credentials should follow Step 3
"identity_assertion + email" below.

## Step 1 — Discover

### 1a. Fetch Protected Resource Metadata

```http
GET https://makeup.land/.well-known/oauth-protected-resource
```

Returns:

```json
{
  "resource": "https://makeup.land/api/v1/",
  "resource_name": "makeup.land V1 API",
  "authorization_servers": ["https://makeup.land"],
  "scopes_supported": ["full", "register", "giftcards", "proposals", "read_only"],
  "bearer_methods_supported": ["header"]
}
```

| Field | Meaning |
|-------|---------|
| `resource` | Root URL of the protected API (the bearer token is sent against this resource). |
| `authorization_servers` | Origins that issue tokens for the above resource. Today makeup.land is its own AS. |
| `scopes_supported` | Available scopes — see Section 5 for what each unlocks. |
| `bearer_methods_supported` | Where the bearer is presented; only `header` is supported. |

### 1b. Fetch Authorization Server Metadata

```http
GET https://makeup.land/.well-known/oauth-authorization-server
```

Returns the `agent_auth` block:

```json
{
  "issuer": "https://makeup.land",
  "agent_auth": {
    "skill": "https://makeup.land/auth.md",
    "register_uri": "mailto:info@makeup.land?subject=API%20access%20request",
    "identity_types_supported": ["identity_assertion"],
    "identity_assertion": {
      "methods": ["email_manual"],
      "credentials_issued": ["api_key"]
    },
    "events_supported": ["credential.issued", "credential.revoked"]
  }
}
```

| Field | Meaning |
|-------|---------|
| `skill` | This file. |
| `register_uri` | Where to initiate credential issuance. Today this is a `mailto:` URI — the agent's operator (or the agent itself, if it can send email) writes to that address with the intended use case. |
| `identity_types_supported` | `identity_assertion` only — no anonymous flow. |
| `identity_assertion.methods` | `email_manual` — the operator provides identity context in an email. |
| `identity_assertion.credentials_issued` | `api_key` — a bearer string with prefix `ml_`. |
| `events_supported` | Lifecycle events the AS may surface in future. No webhook delivery today. |

## Step 2 — Pick a method

There is only one supported method right now: **identity_assertion + email** (Section 3).

## Step 3 — Register (identity_assertion + email)

1. Send an email to [`info@makeup.land`](mailto:info@makeup.land?subject=API%20access%20request) with:
   - The operator's name / company / contact phone.
   - The intended use case (e.g. "Sandra WA agent for order lookups").
   - The desired `scope` (`full` / `register` / `giftcards` /
     `proposals`) and whether `read_only` is acceptable.
   - The originating agent's user-agent string and the IPs / hosts the
     calls will originate from (for rate-limit allowlisting if needed).
2. An admin issues a token against the `api_tokens` table and replies
   with the bearer string (`ml_<hex>`) plus any scope notes.
3. Store the bearer in your secret store. **Never log it.**

Median turnaround: a few business hours during Israel working hours. There
is no SLA today.

## Step 4 — Claim ceremony

Not applicable. Identity is asserted at registration via the email channel
in Step 3. No OTP / out-of-band claim is performed.

## Step 5 — Use the credential

```http
GET https://makeup.land/api/v1/brands
Authorization: Bearer ml_<hex>
```

- Tokens do not expire automatically. They are revoked manually if the
  operator's relationship with makeup.land ends, or on suspected
  compromise (write to the same email).
- Token leakage handling: respond to a `401` by treating the credential
  as invalid; do not auto-retry. Mail `info@makeup.land` for a fresh
  token rather than rotating in-band.
- The phone-keyed endpoints (cart, payment-links, gift-cards list,
  best-deals) do **not** use the bearer; see the OpenAPI spec for the
  query-string `phone` credential they expect instead.

### Idempotency

All mutation endpoints (POST/PATCH/DELETE) accept an optional
`Idempotency-Key` header — a client-generated UUIDv4 per logical
operation. Retries with the same key inside 24 hours replay the original
response.

## Errors

| Code | Where | What to do |
|------|-------|------------|
| 401 `{error_code:"unauthorized"}` | Any API call | Bearer is missing, malformed, or revoked. Stop calling. Mail `info@makeup.land`. |
| 403 `{error_code:"scope_mismatch"}` | Any API call | Token's scope does not permit this endpoint. Request a wider-scope token. |
| 403 `{error_code:"read_only_token"}` | Mutation calls | Token has `read_only=true`. Request a writable token. |
| 429 `{error_code:"rate_limited"}` | Rate-limited endpoint | Honour `Retry-After`. Slow down or request a higher quota. |
| 404 `{error_code:"customer_not_found"}` | Phone-keyed | Phone does not match a customer. Do not retry with the same phone. |
| 409 `{error_code:"insufficient_stock"}` | Cart / order | Read `available` + `requested` and adjust quantity. |
| 409 `{error_code:"insufficient_credits"}` | Credits-mode cart | Wallet balance below price. Drop to ILS tender or top up. |
| 422 `{error_code:"validation_failed"}` | Proposals batch | Inspect `row_errors` array for per-row issues. |
| 5xx | Any | Retry with exponential backoff. Include `Idempotency-Key` on mutations to avoid duplicates. |

The `error_code` field is the stable contract; the human `error`
string copy may change between releases. See the OpenAPI `Error`
component schema for the full enum.

## Revocation

To revoke a token, write to [`info@makeup.land`](mailto:info@makeup.land?subject=API%20access%20request) with
the token's prefix (first 8 chars) and the operator's name. The token is
disabled in `api_tokens` (sets `is_active = false`); subsequent
calls return `401 unauthorized`. No self-service revocation endpoint
exists today.

## References

- This file: [https://makeup.land/auth.md](https://makeup.land/auth.md)
- Protected Resource Metadata: [https://makeup.land/.well-known/oauth-protected-resource](https://makeup.land/.well-known/oauth-protected-resource)
- Authorization Server Metadata: [https://makeup.land/.well-known/oauth-authorization-server](https://makeup.land/.well-known/oauth-authorization-server)
- OpenAPI specification: [https://makeup.land/openapi.json](https://makeup.land/openapi.json)
- Long-form agent guide: [https://makeup.land/llms-full.txt](https://makeup.land/llms-full.txt)
- AI manifest: [https://makeup.land/.well-known/ai-manifest.json](https://makeup.land/.well-known/ai-manifest.json)
- WorkOS auth.md skill format: <https://workos.com/auth-md>
- RFC 9728 (OAuth Protected Resource Metadata): <https://datatracker.ietf.org/doc/rfc9728/>
- RFC 8414 (OAuth Authorization Server Metadata): <https://datatracker.ietf.org/doc/rfc8414/>
