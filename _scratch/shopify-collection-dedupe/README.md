# Shopify Collection Dedupe

One-shot sweep to delete duplicate Shopify collections, picking the keeper by handle match against WooCommerce category slugs (so existing WC→Shopify 301 redirects keep resolving). Approval flow is a Google Sheet, not a CSV.

## Setup

1. **Shopify CLI auth (one-time)** — uses the Shopify CLI's stored auth, not a custom-app token:

   ```sh
   shopify store auth --store techloop-7.myshopify.com --scopes read_products,write_products
   ```

   This opens a browser; approve in the Shopify admin. Token is stored by the CLI and reused on every `shopify store execute` call.

2. **WooCommerce REST creds** — `WooCommerce → Settings → Advanced → REST API`. Read-only key is fine.

3. **Google Sheets access** — `gws` CLI must be authenticated for the Google account that should own the audit sheet:

   ```sh
   gws auth login  # if not already done
   ```

4. **Configure `.env`**:

   ```sh
   cp .env.example .env
   # edit .env with WC key/secret. Shopify needs no token here — CLI auth handles it.
   ```

## Run

```sh
# Pass 1 — audit. Creates a Google Sheet, writes SHEET_ID to .env.
npm run audit

# Open the printed sheet URL. For each `action = review` row, change to keep or delete.
# Optionally edit `suggested_title` for any keeper you want renamed.
# When happy:

# Pass 2 dry-run — prints intended mutations, makes no changes.
npm run execute:dry

# Pass 2 live — runs collectionDelete + collectionUpdate; writes result back to each row.
npm run execute
```

## Verifying

After Pass 2:

1. `npm run audit` again — the new sheet should show zero `delete` rows.
2. Spot-check a deleted collection by GID — Shopify Admin should 404 it.
3. Hit a live WC URL whose category slug matches a kept Shopify handle — the WC 301 should land on the right Shopify collection page.

## How duplicates are detected

- **Smart collections**: grouped by a hash of `(appliedDisjunctively, sorted rules)`. Two smart collections with the same rule set hash are flagged.
- **Manual collections**: grouped by normalised title (lowercase, alphanumeric only).
- Smart and manual collections with the same title will NOT group together — they'll each form their own single-member group (which is excluded from the audit). If you suspect a smart+manual duplicate pair, surface it manually.

## How the keeper is picked

For each duplicate group:
- Build the set of valid WC category slugs (from `/wp-json/wc/v3/products/categories`).
- If exactly one member's `handle` is in that set → it's the keeper; others get `action=delete`.
- If zero or multiple members match → every member gets `action=review`. Human call.

## Sheet columns

| Column | Notes |
|---|---|
| group_id | Synthetic ID per duplicate group (e.g. `smart-3`). |
| collection_id | Shopify numeric ID. |
| handle | Shopify URL handle. |
| title | Shopify display title. |
| products_count | How many products the collection currently has. |
| is_smart | TRUE for rule-based smart collections. |
| rule_set_hash | First 12 chars of the SHA-1 of the rule set. Only set for smart collections. |
| matches_wc_slug | TRUE if `handle` is a valid WC category slug. |
| wc_slug | Same as `handle` when `matches_wc_slug = TRUE`. |
| wc_category_name | WC category display name for that slug. |
| action | `keep` / `delete` / `review`. Dropdown — editable. |
| suggested_title | Auto-suggested cleaner title for keepers. Blank if no suggestion. Edit/blank to control rename. |
| notes | Why this row got its action. |
| result | Filled by `execute.js`: `deleted` / `renamed` / `error: …` |
| result_ts | UTC timestamp of result. |

## Safety notes

- The store is **not live yet**, so deleting a duplicate collection doesn't break any SEO. The WC→Shopify redirect target (the keeper's handle) is the only thing that matters.
- Renaming changes only the `title`, never the `handle` — so even if you rename keepers, redirects still resolve.
- `execute.js` skips any row with a non-empty `result` column, so re-runs are safe / idempotent.
