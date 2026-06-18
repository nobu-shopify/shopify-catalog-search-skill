# Shopify Catalog Search Skill

`catalog-search` is an Agent Skill for searching Shopify's Global Catalog from an LLM/agent. It is designed for demos and uses Shopify's **Catalog MCP** endpoint directly:

```text
https://catalog.shopify.com/api/ucp/mcp
```

The skill uses `search_catalog` to discover candidate products, then refreshes relevant candidates with `get_product` before displaying prices.

## What this skill does

- Searches Shopify's Global Catalog by product keyword.
- Supports official Catalog MCP / UCP schema parameters, including:
  - `query`
  - `like`
  - `context`
  - `signals`
  - `filters.available`
  - `filters.price`
  - `filters.ships_to`
  - `filters.ships_from`
  - `filters.condition`
  - `filters.categories`
  - `filters.shop_ids`
  - `filters.attributes`
  - `filters.rating`
  - `filters.price_tier`
  - `pagination`
- Handles country-local store intent, for example:
  - `日本のストアでAFURIのゆず醤油ラーメンを探して`
  - `Find skincare in stores in Korea`
- For country-local searches, sets both destination and origin by default:

```json
{
  "filters": {
    "ships_to": { "country": "JP" },
    "ships_from": [{ "country": "JP" }]
  }
}
```

## Repository contents

```text
.
├── SKILL.md       # Agent Skill definition and workflow
├── README.md      # This file
├── .env.example   # Environment variable template
└── .gitignore
```

## Requirements

- An LLM/agent runtime that supports Agent Skills, such as Pi.
- `curl`
- `jq`
- Shopify Catalog API access credentials from the Shopify Dev Dashboard.

Optional, for debugging schema only:

- `pnpm`
- `@shopify/ucp-cli`

The skill itself should use direct Catalog MCP JSON-RPC calls for searches, not the UCP CLI.

## Shopify Catalog access and credentials

Shopify Catalog provides select apps and AI agents access to Shopify's product catalog via API and MCP surfaces. Access may require approval.

Relevant Shopify docs:

- Global Catalog MCP reference: https://shopify.dev/docs/agents/catalog/global-catalog
- Catalog overview and Saved Catalogs: https://shopify.dev/docs/agents/catalog
- Dev Dashboard overview: https://shopify.dev/docs/apps/build/dev-dashboard
- Dev Dashboard access token guide: https://shopify.dev/docs/apps/build/dev-dashboard/get-api-access-tokens
- Client credentials: https://shopify.dev/docs/apps/build/authentication-authorization/client-secrets
- Shopify Catalog changelog: https://shopify.dev/changelog/shopify-catalog

### Create or locate Catalog credentials

According to the Shopify developer documentation, Catalog credentials are obtained from the **Catalog** section of the Dev Dashboard.

1. Open the Shopify Dev Dashboard:

   ```text
   https://dev.shopify.com/dashboard
   ```

2. Sign in with the Shopify organization that has Catalog access.

3. Open **Catalogs** from the Dev Dashboard navigation.

4. Create a catalog or open an existing saved catalog.

   Saved Catalogs let you configure a reusable product discovery scope. You can filter or constrain the catalog by inputs such as product taxonomy, price ranges, specific shops, and other saved parameters. The Dev Dashboard also provides a preview panel for testing the configuration before saving.

5. In the catalog/app access area, copy your API credentials:

   - Client ID
   - Client Secret

6. Keep the Client Secret secure. Do not commit it to Git.

> Note: Some Catalog access is gated. If you do not see Catalogs or API credentials in your Dev Dashboard, your organization may need Catalog access enabled.

### Configure local environment variables

Create a local `.env` file or export the variables in your shell profile.

For shell profile setup:

```bash
# ~/.zshrc or ~/.bashrc
export SHOPIFY_CATALOG_CLIENT_ID="paste-your-client-id-here"
export SHOPIFY_CATALOG_CLIENT_SECRET="paste-your-client-secret-here"
```

Then reload your shell:

```bash
source ~/.zshrc
```

Or copy the example file:

```bash
cp .env.example .env
```

Then fill in:

```bash
SHOPIFY_CATALOG_CLIENT_ID="..."
SHOPIFY_CATALOG_CLIENT_SECRET="..."
```

### Fetch an access token

Catalog API credentials use the client credentials grant. Tokens expire, so the skill fetches a fresh token per search.

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

Use it with Catalog MCP requests:

```bash
-H "Authorization: Bearer $TOKEN"
```

## How the skill calls Catalog MCP

The skill calls the Catalog MCP endpoint directly with JSON-RPC:

```bash
curl --silent --show-error --request POST 'https://catalog.shopify.com/api/ucp/mcp' \
  -H 'Content-Type: application/json' \
  -H "Authorization: Bearer $TOKEN" \
  --data '{
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
          "query": "AFURI 柚子醤油 らーめん",
          "context": {
            "address_country": "JP",
            "currency": "JPY",
            "language": "ja-JP"
          },
          "filters": {
            "available": true,
            "ships_to": { "country": "JP" },
            "ships_from": [{ "country": "JP" }]
          },
          "pagination": { "limit": 10 }
        }
      }
    }
  }'
```

Important schema detail:

```json
"ships_to": { "country": "JP" }
```

is an object, while:

```json
"ships_from": [{ "country": "JP" }]
```

is an array of objects.

## Current demo workaround: do not display `search_catalog` prices

For demos, this skill intentionally does **not** use `search_catalog` prices for final display.

Observed issue:

- `search_catalog` can return inconsistent product summary prices for some products, especially mixed currency values in `price_range` and `variants[].price`.
- `get_product` can return the correct refreshed product price for the same product.

Therefore the skill follows this workflow:

1. Call `search_catalog` for discovery and ranking.
2. Extract the top relevant product IDs.
3. Call `get_product` for each top candidate.
4. Display prices from `get_product.price_range` or `get_product.variants[].price` only.

In other words:

```text
search_catalog = candidate discovery
get_product    = user-facing product details and prices
```

If `search_catalog` and `get_product` disagree, show the `get_product` price.

## Install the skill in Pi

Clone the repository into your Pi skills directory:

```bash
mkdir -p ~/.pi/agent/skills
cd ~/.pi/agent/skills
git clone https://github.com/nobu-shopify/shopify-catalog-search-skill.git catalog-search
```

Pi discovers skill directories containing `SKILL.md`. After restarting Pi, the skill should appear as `catalog-search`.

If skill commands are enabled, you can call:

```text
/skill:catalog-search AFURI 柚子醤油 らーめん
```

You can also use natural language:

```text
Shopify Catalogで、日本のストアでAFURIのゆず醤油ラーメンを探して
```

```text
Shopify Catalogで、日本のストアでAFURIの辛いラーメンを探して
```

```text
Find skincare in stores in Korea, 4 stars and up
```

## Example prompts

### Japanese domestic search

```text
Shopify Catalogで、日本のストアでこってり醤油ラーメンを探して
```

The skill will infer:

```json
{
  "context": {
    "address_country": "JP",
    "currency": "JPY",
    "language": "ja-JP"
  },
  "filters": {
    "available": true,
    "ships_to": { "country": "JP" },
    "ships_from": [{ "country": "JP" }]
  }
}
```

### Korea-local search

```text
Find skincare in stores in Korea, 4 stars and up
```

The skill will infer:

```json
{
  "context": {
    "address_country": "KR",
    "currency": "KRW",
    "language": "ko-KR"
  },
  "filters": {
    "available": true,
    "ships_to": { "country": "KR" },
    "ships_from": [{ "country": "KR" }],
    "rating": {
      "variant": { "min": 4 }
    }
  }
}
```

### Shipping-only search

```text
Find waterproof jackets that ship to Canada under $150
```

The skill will set only destination shipping unless store origin is also requested:

```json
{
  "filters": {
    "available": true,
    "ships_to": { "country": "CA" },
    "price": { "max": 15000 }
  }
}
```

## Notes and caveats

- Do not commit credentials.
- Always fetch fresh results. Do not cache Catalog search results or images for demos.
- Treat `context.currency` as a localization signal, not a strict currency filter.
- If the user explicitly asks for a currency, post-filter results and/or rely on `get_product` prices.
- `search_catalog` results are useful for discovery, but final product prices should come from `get_product`.
- The buyer-facing PDP is `variants[].url`.
- The buy-now handoff URL is `variants[].checkout_url`.
- The seller identity for downstream cart/checkout is `variants[].seller.domain`.

## Development / debugging

To inspect the live accepted schema, you can use the UCP CLI for debugging only:

```bash
pnpm dlx @shopify/ucp-cli catalog search --input-schema --format json
```

But for the skill's actual demo searches, use the direct Catalog MCP endpoint.

## License

This repository does not currently declare a license. Add one before distributing outside your organization.
