---
name: makeup-land-shade-match-catalog
description: Find makeup.land products by target shade (CIE ΔE 2000 match on a hex colour), hue family, brand or cross-lingual natural-language query, and read the ILS and ℳ-credit prices side by side. Anonymous — no token needed.
api: makeup.land V1 API (https://makeup.land/api/v1) and MCP tool list_products (https://makeup.land/api/mcp)
operations:
  - listProducts
  - listBrands
method: generated
generated: '2026-09-19'
source: openapi/makeup-land-openapi.yml, https://makeup.land/llms-full.txt ("Search semantics"), mcp/makeup-land-mcp-tools.json
---

# Shade-match and search the makeup.land catalog

## When to use
A shopper asks for "a lipstick like #C2185B", "a warm foundation near #E8D4B8", "the most popular NYX products under ₪150", or any category lookup in Hebrew, English or another language.

## Auth
- `GET /api/v1/products` (`listProducts`) is **anonymous** for `q`, `tag`, `brand`, `near_hex`, `hue_family`, `sort`, `limit`, `page`.
- It becomes bearer-only the moment you pass `phone=`, `relevant_to_phone=` or `include=inventory` (PII-keyed projections). `GET /api/v1/brands` (`listBrands`) always needs `Authorization: Bearer ml_<hex>` — see `makeup-land-auth.md`.
- MCP: call tool `list_products` with the same arguments; `list_brands` needs the bearer.

## Steps
1. **Pick the filter.** The API rejects a bare call — send at least one of `q`, `tag`, `brand`, `near_hex`, or `sort=popularity|rating` (observed 400 otherwise). `hue_family` is a refinement and must be paired with one of those.
2. **Shade match:** `GET /api/v1/products?near_hex=%23C2185B&limit=5`. Each product returns `shade_match: {hex, delta_e}` — the closest variant swatch and its perceptual distance. Default `delta_e_max` is 80 for a single hex; pass up to 8 hexes as CSV for palette coverage with `coverage=all|any` (default `delta_e_max` 200 in multi-hex mode).
3. **Refine by undertone/hue:** add `hue_family=` one of the API's families (`white black gray red coral orange brown nude beige yellow green blue purple pink` per llms-full.txt; the MCP tool enum is `red orange yellow green blue purple pink brown neutral`).
4. **Natural-language lookup:** prefer `q=` (cross-lingual semantic search — `lipstick`, `שפתון`, `lápiz labial` all work) over `tag=`, which is an EXACT match against Hebrew-stored tags (`tag=שפתון`), case-insensitive and trimmed.
5. **Rank and page:** `sort=price_asc|price_desc|popularity|rating|relevance` (`rating` is Bayesian-shrunk). Page with `limit` (1–50) and `page` (1-indexed); stop when `has_more` is false, but if `partial` is true the next page may still hold matches after post-filtering.
6. **Read prices correctly:** `price` / `compare_at_price` are decimal ILS (`compare_at_price > price` means on sale); `credit_price` is integer agorot in ℳ-credits and `null` means ILS-only. Product `tags` come back lower-case Hebrew.
7. **Brands:** `GET /api/v1/brands` (bearer) returns `name`, `name_he`, `slug`, `product_count`; use `slug` as `brand=` on listProducts.

## Errors
`400 invalid_parameter` (bad hex, missing required filter), `401 unauthorized` (a PII parameter without a bearer). Branch on `error_code`; the `error` string may be Hebrew or English.

## Do not
- Do not send an English category word as `tag=` — use `q=`.
- Do not pass `phone=` without a bearer; it is a customer selector, not a credential.
