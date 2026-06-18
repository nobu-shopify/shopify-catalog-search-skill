---
name: catalog-search
description: "Search Shopify's Global Catalog for products by keyword, price, shipping destination/origin, merchant, category, attributes, ratings, condition, or country-local store intent. Use when the user wants to find products, search the Shopify catalog, look up items, browse merchandise, or asks for stores in a specific country such as Japanese stores or stores in Korea."
argument-hint: "<search query> [optional: country-local stores, ships to/from <country>, under/over <amount>, category, attributes, rating]"
---

# Shopify Catalog Search

**Purpose:** Search Shopify's Global Catalog by directly calling the Catalog MCP JSON-RPC endpoint (`search_catalog`). This skill is for demos, so always use the direct Catalog MCP call path rather than the UCP CLI for actual searches.

**Triggers:** `/catalog-search`, "search catalog for", "find products", "look up products", "catalog search", "Shopify Catalogで", "日本のストアで", "stores in <country>"

**References:**

- Global Catalog MCP: https://shopify.dev/docs/agents/catalog/global-catalog
- Global Catalog extension filters: https://shopify.dev/docs/agents/catalog/global-catalog-extension
- Catalog overview: https://shopify.dev/docs/agents/catalog

---

## Setup (one-time, per user)

Export Catalog API credentials in your shell profile (`~/.zshrc`) so they are available to all tools:

```bash
# ~/.zshrc
export SHOPIFY_CATALOG_CLIENT_ID="paste-your-client-id-here"
export SHOPIFY_CATALOG_CLIENT_SECRET="paste-your-client-secret-here"

# Optional: saved Catalog MCP URL copied from Dev Dashboard > Catalogs.
# If omitted, the skill uses the default Global Catalog MCP endpoint.
export SHOPIFY_CATALOG_MCP_URL="https://catalog.shopify.com/api/ucp/mcp"
```

Then reload your shell: `source ~/.zshrc`

Dev Dashboard docs refer to Saved Catalogs exposing a custom endpoint URL from the Catalogs **Access** area. Older workflows sometimes called the identifier in that URL a catalog slug/blob ID. For this skill, configure the full copied URL as `SHOPIFY_CATALOG_MCP_URL` when you want to use a saved Catalog. If omitted, use the default Global Catalog MCP endpoint.

`SHOPIFY_CATALOG_SLUG` is only needed for the legacy `discover.shopifyapps.com/global/v2/search/<slug>` GET endpoint. Do not use the legacy endpoint for this skill unless the user explicitly asks to compare legacy behavior; for normal demo searches, always call the Catalog MCP endpoint directly.

**To get credentials:**
1. Go to https://dev.shopify.com/dashboard → sign in
2. Navigate to **Catalogs** in the left nav
3. Create a Catalog or request Global Catalog access as needed
4. Copy your **Client ID** and **Client Secret** from the API Keys tab

---

## Workflow

### Step 1: Parse Request

Extract as many of these official UCP / MCP schema fields as the user's request supports:

| User intent | Official field | Type / example | Notes |
|---|---|---|---|
| Search terms | `catalog.query` | `"こってり醤油ラーメン"` | Free-text product query. Required unless using `like`. |
| Similar product/image search | `catalog.like` | `[{"id":"gid://shopify/p/..."}]` or image object | 1–2 items. Use when user says "similar to this" and provides an ID/image. |
| Buyer country | `catalog.context.address_country` | `"JP"` | Soft signal for localization/ranking. |
| Buyer region | `catalog.context.address_region` | `"Tokyo"` | Soft signal. |
| Buyer postal code | `catalog.context.postal_code` | `"100-0001"` | Soft signal / delivery estimate fidelity. |
| Language | `catalog.context.language` | `"ja-JP"` | Soft signal. |
| Preferred currency | `catalog.context.currency` | `"JPY"` | Soft signal; post-filter by currency only when user asks. |
| Buyer intent | `catalog.context.intent` | `"gift under ¥5000"` | Free-text background. |
| Buyer IP signal | `catalog.signals.dev.ucp.buyer_ip` | IP string | Only if explicitly available; do not invent. |
| User agent signal | `catalog.signals.dev.ucp.user_agent` | UA string | Only if explicitly available; do not invent. |
| Availability | `catalog.filters.available` | `true` | Default true; set false only when user wants unavailable/out-of-stock included. |
| Price range | `catalog.filters.price.min/max` | `{ "min": 5000, "max": 20000 }` | Minor currency units. |
| Shipping destination | `catalog.filters.ships_to` | `{ "country":"JP" }` | Hard filter. Country is ISO 3166-1 alpha-2; optional `region`, `postal_code`. |
| Shipping origin / store country | `catalog.filters.ships_from` | `[{ "country":"JP" }]` | Hard filter. **Array**, not object. Multiple entries use OR logic. |
| Product condition | `catalog.filters.condition` | `["new"]` | Known values: `new`, `secondhand`. Multiple values use OR. |
| Shopify taxonomy categories | `catalog.filters.categories` | `["gid://shopify/TaxonomyCategory/..."]` | Use taxonomy IDs when known. Plain category words usually belong in `query` unless an ID is known. |
| Shop restriction | `catalog.filters.shop_ids` | `["gid://shopify/Shop/..."]` | Restrict to specific shop GIDs when known. |
| Product attributes | `catalog.filters.attributes` | `[{"name":"Color","values":["Red"]}]` | Supported names include `Color`, `Size`, `Target gender`. Entries are AND; values inside an entry are OR. |
| Rating threshold | `catalog.filters.rating.variant.min` | `{ "variant": { "min": 4.0 } }` | 0–5 scale. |
| Rating count threshold | `catalog.filters.rating.variant.min_count` | `{ "variant": { "min_count": 10 } }` | Minimum review count. |
| Relative price tier | `catalog.filters.price_tier` | `["low"]` | Known values: `low`, `medium`, `high`. |
| Page size | `catalog.pagination.limit` | `10` | Default 10; minimum 1. |
| Next page | `catalog.pagination.cursor` | opaque cursor | Use only from response CTA; do not hand-roll. |
| Predefined output shape | `catalog.view` | string | Official schema field: predefined response shape. When using UCP CLI, `--view` is also available for JMESPath projection. Not part of product filtering. |

If no query or `like` input is provided, ask: "What would you like to search for in the Shopify Catalog?"

Check that `SHOPIFY_CATALOG_CLIENT_ID` and `SHOPIFY_CATALOG_CLIENT_SECRET` are set. If missing, list which ones are unset and remind the user to follow Setup.

### Country-local store intent

When the user asks for stores **in a specific country**, interpret it as a country-local store search. Examples:

- Japanese: **「日本のストアで」**, **「日本国内のストア」**, **「国内ストア」**, **「日本発送」**, **「国内発送」**, **「日本出荷」**
- English: **"Japanese stores"**, **"stores in Japan"**, **"in stores in Korea"**, **"stores in South Korea"**, **"Korean stores"**, **"shops in France"**

For country-local store intent, automatically apply:

```json
{
  "catalog": {
    "query": "<query>",
    "context": {
      "address_country": "<COUNTRY_CODE>",
      "currency": "<LOCAL_CURRENCY_IF_KNOWN>",
      "language": "<LANGUAGE_IF_KNOWN>",
      "intent": "Find products from stores in <country> that can ship domestically."
    },
    "filters": {
      "available": true,
      "ships_to": { "country": "<COUNTRY_CODE>" },
      "ships_from": [{ "country": "<COUNTRY_CODE>" }]
    },
    "pagination": { "limit": 10 }
  }
}
```

Key rule: when the user says **"stores in <country>"** or equivalent, default both `filters.ships_to.country` and `filters.ships_from[0].country` to that country. If the user only says **"ships to <country>"**, set only `ships_to`. If the user only says **"ships from <country>"**, set only `ships_from`.

For Japan, also render in Japanese and prefer/post-filter JPY results by default because Japanese domestic-shopping intent commonly implies Japanese merchants and JPY pricing:

- Set `context.address_country=JP`, `context.currency=JPY`, `context.language=ja-JP`.
- Set `filters.ships_to={"country":"JP"}` and `filters.ships_from=[{"country":"JP"}]`.
- Keep main results only when they are available, JPY-priced, and have Japan-domestic signals.
- Japan-domestic signals include Japanese shop name, Japanese title/description, `.jp` or Japan-focused domain, `日本出荷`, `国内発送`, Japanese address/context, and JPY prices.
- Do **not** claim definitive store location unless the response proves it. Say **「日本国内ストアと思われる」** and show the reason.

For other countries, use local currency/language as context when obvious (e.g. KR/KRW/ko-KR, US/USD/en-US, GB/GBP/en-GB, CA/CAD/en-CA, AU/AUD/en-AU, FR/EUR/fr-FR, DE/EUR/de-DE). Do not hard-exclude non-local currency unless the user explicitly asks for that currency only; instead separate uncertain results if needed.

---

### Step 2: Get Bearer Token

```bash
TOKEN=$(curl --silent --request POST \
  --url 'https://api.shopify.com/auth/access_token' \
  -H 'Content-Type: application/json' \
  --data "{
    \"client_id\": \"${SHOPIFY_CATALOG_CLIENT_ID}\",
    \"client_secret\": \"${SHOPIFY_CATALOG_CLIENT_SECRET}\",
    \"grant_type\": \"client_credentials\"
  }" | jq -r '.access_token')
```

Tokens expire after 60 minutes. Always fetch a fresh token per search — do not cache.

If the token is null or empty, report: "Auth failed — verify SHOPIFY_CATALOG_CLIENT_ID and SHOPIFY_CATALOG_CLIENT_SECRET."

---

### Step 3: Execute Search by directly calling Catalog MCP

Always use the direct Catalog MCP endpoint for searches in this skill. Use a saved Catalog URL from Dev Dashboard when configured; otherwise use the default Global Catalog MCP endpoint:

```bash
MCP_URL="${SHOPIFY_CATALOG_MCP_URL:-https://catalog.shopify.com/api/ucp/mcp}"
```

Do **not** use `ucp catalog search` for demo searches. The UCP CLI is acceptable only for debugging/schema inspection, not for producing the user's search results. Send a JSON-RPC `tools/call` request to the Catalog MCP `search_catalog` tool:

```bash
curl --silent --show-error --request POST "$MCP_URL" \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  --data @- <<'JSON' | jq .
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "id": 1,
  "params": {
    "name": "search_catalog",
    "arguments": {
      "meta": {
        "ucp-agent": {
          "profile": "https://shopify.dev/ucp/agent-profiles/2026-04-08/valid-with-capabilities.json"
        }
      },
      "catalog": {
        "query": "<QUERY>",
        "context": {
          "address_country": "<COUNTRY_CODE_IF_KNOWN>",
          "currency": "<CURRENCY_IF_KNOWN>",
          "language": "<LANGUAGE_IF_KNOWN>",
          "intent": "<BUYER_INTENT_IF_USEFUL>"
        },
        "filters": {
          "available": true,
          "ships_to": { "country": "<DESTINATION_COUNTRY>" },
          "ships_from": [{ "country": "<ORIGIN_COUNTRY>" }]
        },
        "pagination": { "limit": 10 }
      }
    }
  }
}
JSON
```

Only include fields that are known or inferred from the user request. Do not include placeholder values in actual requests. If using the official `catalog.view` field, put it inside `catalog`; if using the UCP CLI `--view` option, pass it as a CLI option instead.

#### Schema/debugging only

For demos and user-facing searches, always call Catalog MCP directly as shown above. If you need to debug schema validation or double-check the live accepted schema, you may use the UCP CLI only for inspection:

```bash
pnpm dlx @shopify/ucp-cli catalog search --input-schema --format json
```

After inspection, return to the direct Catalog MCP JSON-RPC endpoint for the actual search.

---

### Step 4: Refresh candidate details before formatting

Demo workaround: **do not display prices from `search_catalog`**. `search_catalog` is useful for ranking and discovery, but its summary `price_range` can be stale or internally inconsistent for some products. Before showing user-facing prices, call Catalog MCP `get_product` for the top relevant candidates and use `get_product`'s `price_range` / `variants[].price` as the source of truth.

Use `search_catalog` only for:

- candidate product IDs
- titles/descriptions for rough relevance
- seller/PDP/checkout URLs when `get_product` does not provide a better value

Use `get_product` for:

- final displayed price and currency
- final variant-level price
- final options/availability where available

If `search_catalog` and `get_product` disagree, mention that prices were refreshed via `get_product` and display the `get_product` price.

### Step 5: Format Results

Official MCP responses use snake_case fields such as:

- `result.structuredContent.products[]`
- `price_range.min.amount`, `price_range.min.currency`
- `variants[].available_for_sale`
- `variants[].seller.domain`, `variants[].seller.url`
- `variants[].url` for PDP
- `variants[].checkout_url` for buy-now handoff

Render each product card:

```markdown
**<title>** — <formatted price>
<description, max 2 sentences>
Seller: <variants[0].seller.domain or seller.url>
Rating: <rating.value>/5 (<rating.count> reviews) | Options: <options summary>
→ [View Product](<variants[0].url>)  [Buy Now](<variants[0].checkout_url>)
```

For Japan-local mode, render in Japanese:

```markdown
**<title>** — ¥<price>
<description, max 2 sentences>
ストア: <seller domain/url or shop name if present>
国内ストア判定: <ships_from=JP / JPY価格 / 日本語商品ページ / .jpドメイン / 日本出荷表記など>
Rating: <rating.value>/5 (<rating.count> reviews) | Options: <options summary>
→ [商品を見る](<variants[0].url>)  [Buy Now](<variants[0].checkout_url>)
```

Formatting rules:

- Amounts are in minor currency units. Divide by 100 for most currencies; render JPY with no decimals when appropriate.
- Use the returned currency; do not assume USD.
- If min and max price are the same, show one price.
- If no variants are sale-ready, exclude from main results unless the user asked to include unavailable items.
- Show total count / pagination information when present. If the response CTA includes a next-page command, mention that more results are available.
- For country-local results, include why each result matched the local-country criteria. If confidence is low, put it under **"Country/store not fully confirmed but close candidates"**.

---

### Step 6: Error Handling

| Scenario | Action |
|---|---|
| Missing env vars | List unset credentials and link to Setup. |
| Token fetch fails | "Auth failed — check CLIENT_ID and CLIENT_SECRET." |
| HTTP 401 | "Token rejected — try again or re-check credentials." |
| HTTP 404 on `discover.shopifyapps.com` POST | Explain that official MCP JSON-RPC uses `https://catalog.shopify.com/api/ucp/mcp`, not the legacy GET endpoint. |
| `SCHEMA_VALIDATION_FAILED` | Run or suggest `ucp catalog search --input-schema --format json`; fix shapes such as `ships_from` needing an array. |
| Empty products array | "No products found for '<query>'. Try broader or different terms." |
| API unreachable / 5xx | Report the error code and suggest retrying. |
| `jq` not installed | Suggest: `brew install jq`. |
| Country-local request returns mixed results | Keep high-confidence local results in the main section; put uncertain candidates in a separate caveated section. |
| `search_catalog` price/currency conflicts with `get_product` or storefront | Treat `search_catalog` price as non-authoritative; refresh via `get_product` and show only the refreshed price. |

---

## Filter Reference / Natural Language Mapping

| What the user says | Official schema | Example |
|---|---|---|
| "under $50" | `catalog.filters.price.max` | `5000` minor units |
| "over $100" | `catalog.filters.price.min` | `10000` minor units |
| "$20–$80" | `catalog.filters.price.min/max` | `2000`–`8000` |
| "available only" | `catalog.filters.available=true` | default |
| "include out of stock" | `catalog.filters.available=false` | includes unavailable items |
| "ships to Canada" | `catalog.filters.ships_to.country=CA` | destination Canada |
| "ships from Korea" | `catalog.filters.ships_from=[{"country":"KR"}]` | origin Korea |
| "in stores in Korea" | `ships_to=KR` + `ships_from=[{"country":"KR"}]` | country-local store intent |
| "日本のストアで" | `ships_to=JP` + `ships_from=[{"country":"JP"}]` + JP context | Japan-local mode |
| "JPYで", "円建て" | `context.currency=JPY` + post-filter returned JPY | currency filter when explicitly requested |
| "new only" | `catalog.filters.condition=["new"]` | new products |
| "secondhand" | `catalog.filters.condition=["secondhand"]` | used/secondhand products |
| "red size M" | `catalog.filters.attributes` | `Color=Red`, `Size=M` |
| "4 stars and up" | `catalog.filters.rating.variant.min=4` | rating threshold |
| "at least 10 reviews" | `catalog.filters.rating.variant.min_count=10` | review count threshold |
| "cheap / budget" | `catalog.filters.price_tier=["low"]` | relative price tier |
| "premium" | `catalog.filters.price_tier=["high"]` | relative price tier |
| "show 5" / "top 5" | `catalog.pagination.limit=5` | page size |
| "next page" | `catalog.pagination.cursor` | use response CTA cursor only |

---

## Examples

**`/catalog-search running shoes under $150 ships to US show 5`**

```json
{
  "catalog": {
    "query": "running shoes",
    "filters": {
      "available": true,
      "price": { "max": 15000 },
      "ships_to": { "country": "US" }
    },
    "pagination": { "limit": 5 }
  }
}
```

**`/catalog-search 日本のストアでこってり醤油ラーメン`**

```json
{
  "catalog": {
    "query": "こってり醤油ラーメン",
    "context": { "address_country": "JP", "currency": "JPY", "language": "ja-JP" },
    "filters": {
      "available": true,
      "ships_to": { "country": "JP" },
      "ships_from": [{ "country": "JP" }]
    },
    "pagination": { "limit": 10 }
  }
}
```

**`Find skincare in stores in Korea, 4 stars and up`**

```json
{
  "catalog": {
    "query": "skincare",
    "context": { "address_country": "KR", "currency": "KRW", "language": "ko-KR" },
    "filters": {
      "available": true,
      "ships_to": { "country": "KR" },
      "ships_from": [{ "country": "KR" }],
      "rating": { "variant": { "min": 4 } }
    },
    "pagination": { "limit": 10 }
  }
}
```

**`Find secondhand red jackets from shops in France under €80`**

```json
{
  "catalog": {
    "query": "jackets",
    "context": { "address_country": "FR", "currency": "EUR", "language": "fr-FR" },
    "filters": {
      "available": true,
      "condition": ["secondhand"],
      "price": { "max": 8000 },
      "ships_to": { "country": "FR" },
      "ships_from": [{ "country": "FR" }],
      "attributes": [{ "name": "Color", "values": ["Red"] }]
    },
    "pagination": { "limit": 10 }
  }
}
```

---

## Notes

- **Do not cache** search results or images. Fetch fresh results in real time.
- Always use the direct Catalog MCP JSON-RPC endpoint for this demo skill's actual searches; do not use the UCP CLI for user-facing search results.
- `ships_to` is an object; `ships_from` is an array of objects.
- The seller identity for follow-up cart/checkout is `variants[M].seller.domain`.
- The buyer-facing PDP is `variants[M].url`; buy-now handoff is `variants[M].checkout_url`.
- Demo workaround: never use `search_catalog` prices for final display. Always refresh relevant candidates with `get_product` before rendering prices.
- If a user wants a specific product after search, use `get_product` with the product or variant ID via UCP CLI/MCP rather than reusing stale variant data.
