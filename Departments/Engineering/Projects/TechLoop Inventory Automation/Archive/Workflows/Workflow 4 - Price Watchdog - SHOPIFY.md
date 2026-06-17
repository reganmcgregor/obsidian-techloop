---
notion_page_id: null
notion_parent_id: null
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: context
synced_at: null
---

# Workflow 4 — Price Watchdog (Shopify)

**Status:** Live  
**n8n Workflow ID:** `yExNIbiIGVFOoJ4N`  
**Workflow Name:** `TL_Price_Watchdog`  
**Schedule:** Daily at 6:00 AM (cron: `0 6 * * *`)  
**Rebuilt:** 2026-06-18 (Phase 12)  
**Predecessor:** [[Workflow 4 - Price Watchdog - CREATED]] (WC-based, frozen 2026-05-05)

---

## Purpose

Safety-net monitor that runs every morning and alerts on products selling below cost or with dangerously low margins (< 10%). Sends a Slack message to `#price-watchdog` with one-click actions per product.

---

## Architecture

```
Schedule Trigger (6 AM daily)
  → Capture Start Time
  → Get Low Margin Products   [Postgres: SELECT * FROM vw_low_margin_products]
  → Check for Alerts          [IF: count > 0]
      → Format Slack Blocks   [Code: builds one item per product + 1 summary item]
          → Send Slack        [Slack: #price-watchdog, C0ADF089R3M — sends N messages, one per item]
```

---

## Data Source — `vw_low_margin_products`

**Replaces** the WC-era view (which joined `tl_wc_products_mirror`).

**Tables:**
- `tl_shopify_products_mirror p` — supplier name, cost, status, Shopify GIDs
- `tl_shopify_variants_mirror v` — live price, variant GID, SKU
- `tl_feed_leader_raw l` — RRP (joined via `p.supplier_product_id = l.stock_code`)

**Filter:** `TRIM(p.supplier_name) = 'Leader'`, `p.status = 'ACTIVE'`, `v.price > 0`, `p.supplier_cost_inc_gst > 0`, margin < 15% (pre-filter to catch WARNING + CRITICAL both)

**Alert levels:**
- `CRITICAL`: `supplier_cost_inc_gst > v.price` (selling below cost)
- `WARNING`: margin < 10%

**Key columns returned:** `shopify_id`, `product_gid`, `variant_gid`, `sku`, `name`, `current_price`, `cost`, `leader_rrp`, `margin_percent`, `alert_level`

---

## Slack Message Format

Each run sends **N+1 separate Slack messages** to `#price-watchdog`:

**Message 1 — Summary header:**
```
[header]   🚨 Price Watchdog Alert
[section]  Critical: X  |  Warning: X  |  Total Issues: X  |  Showing top 12
[context]  Synced: HH:MM AM AEST
```

**Messages 2–13 — One per product (up to 12):**
```
[section]  🔴 *SKU* — Product Name
           Price: $X | Cost: $X | Margin: X% | RRP: $X
[actions]  [Edit in Shopify]  [Set to RRP $X.XX]
```

- Max 12 products shown as individual cards (no block-limit concern — 2 blocks per card)
- Clicking "Set to RRP" replaces **only that card** with a confirmation (`replace_original: true` targets the specific message)

**"Edit in Shopify" button:** URL button → `https://admin.shopify.com/store/techloop-7/products/{shopify_id}`

**"Set to RRP $X.XX" button:** Interactive button (primary style)
- `action_id: "set_price_to_rrp"`
- `value`: compact JSON `{"vGid":"...","pGid":"...","rrp":"X.XX","sku":"..."}`
- Handled by `TL_Slack_Interaction_Handler_Shopify` (`hikIeVV081e76pEv`)
- On success: card replaced with `:white_check_mark: Price updated to $X.XX (RRP) for *SKU* by @user`

---

## Interactive Button Handler

The `set_price_to_rrp` action is routed through the central Slack dispatcher:

**`TL_Slack_Interaction_Handler_Shopify`** → Switch case 10 → `set_price_to_rrp` branch:

```
Set Price Get Token   [HTTP POST → Shopify OAuth token endpoint]
  → Update Price in Shopify  [HTTP POST → Shopify Admin GraphQL]
      mutation productVariantsBulkUpdate($productId: ID!, $variants: [...])
      Sets variant price to leader_rrp (string, e.g. "299.00")
  → Update Variants Mirror   [Postgres: UPDATE tl_shopify_variants_mirror SET price = rrp]
  → Confirm Price Updated    [HTTP POST → response_url (replace_original)]
      ":white_check_mark: Price updated to $X.XX (RRP) for *SKU* by @user"
```

**Auth:** Shopify Client Credentials (`4Kf8ZBknzZl6bvvi`) → POST client_credentials grant → 24h access token  
**API endpoint:** `https://techloop-7.myshopify.com/admin/api/2025-07/graphql.json`

---

## Credentials

| Credential | n8n ID | Used for |
|---|---|---|
| Supabase Postgres | `BSoGuZ9BOv4OWqXf` | `vw_low_margin_products` query + mirror update |
| TechLoop Automation (Slack) | `DvtYgxYgI1p5Q3QX` | Send alert to `#price-watchdog` |
| Shopify Client Credentials | `4Kf8ZBknzZl6bvvi` | Token fetch for GraphQL mutation |

---

## Key Differences from WC Version

| Aspect | WC (FROZEN) | Shopify (LIVE) |
|---|---|---|
| View source | `tl_wc_products_mirror` | `tl_shopify_products_mirror` + `tl_shopify_variants_mirror` |
| Status filter | `status = 'publish'` | `status = 'ACTIVE'` |
| Supplier filter | `TRIM(acf_supplier_name) = 'Leader'` | `TRIM(supplier_name) = 'Leader'` |
| Price field | `w.price` | `v.price` (variant level) |
| Cost field | `l.dbp_inc_gst` (from feed) | `p.supplier_cost_inc_gst` (denormalised into mirror) |
| Product link | WP Admin edit URL | Shopify Admin URL |
| Edit button | "Edit Product" → WP admin | "Edit in Shopify" → Shopify admin |
| RRP button | None | "Set to RRP $X.XX" → GraphQL price update |

---

## Related

- **[[ONE-OFF] TL_Bulk_RRP_Update** (`meFlEpSpYdAw7cmN`) — one-off webhook-triggered workflow that bulk-sets all low-margin products to their leader RRP in Shopify + mirrors. Run once to correct historical pricing; trigger manually via webhook.

---

## Notes

- `leader_rrp` can be null for some products (RRP button is hidden when null/zero)
- The view's 15% pre-filter (vs 10% alert threshold) exists to surface near-warning products without noise — only CRITICAL and WARNING rows appear in alerts, but borderline products are visible in the raw view
- Slack button value is ~115 chars (well within 255-char limit) for typical Shopify GIDs
- The `compare_at_price` field (Shopify's "was" price) is returned by the view but not modified by the RRP button — only `price` is updated
