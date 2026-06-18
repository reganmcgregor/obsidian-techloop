---
status: done
notion_page_id: 3568c3d3-c03e-815c-8c35-cacd6587a650
notion_parent_id: 3558c3d3-c03e-8164-9b2c-ff4243c5147a
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# Phase 12 — Workflow Rebuild Tier 4 (Alerts & Utilities)

**Completed:** 2026-06-18  
**See also:** [[TechLoop Shopify Migration]], [[Phase 11 — Workflow Rebuild Tier 3 (Enrichment)]], [[Workflow 4 - Price Watchdog - SHOPIFY]]

Rebuild utility and alerting workflows for Shopify: `TL_Price_Watchdog` rewired to query Shopify pricing + one-click RRP button, and `TL_Queue_Reviewer_Shopify` reactivated.

---

## Scope

| Workflow | Status | Notes |
|---|---|---|
| TL_Price_Watchdog | ✅ Rebuilt (2026-06-18) | Shopify data source + interactive "Set to RRP" button |
| TL_Queue_Reviewer_Shopify | ✅ Already active (2026-05-31) | No rewrite needed — queue table is WC-agnostic |
| [ONE-OFF] Bulk RRP Update | ✅ Run (2026-06-18) | CLI script; 333 products corrected |

---

## WI-A — TL_Price_Watchdog

**n8n ID:** `yExNIbiIGVFOoJ4N`  
**Schedule:** Daily at 6:00 AM (`0 6 * * *`)

### What changed from the WC version

The WC version (`[FROZEN-2026-05] TL_Price_Watchdog`) queried `tl_wc_products_mirror` and linked to WP Admin. The Shopify rebuild:

- Re-pointed the Postgres node to `vw_low_margin_products` (Shopify mirror tables — see below)
- Changed "Edit Product" URL button to "Edit in Shopify" → `https://admin.shopify.com/store/techloop-7/products/{shopify_id}`
- Added an interactive **"Set to RRP $X.XX"** button per product card
- Redesigned from one combined Slack message to **N+1 individual messages** (1 summary + up to 12 product cards), eliminating Slack's 50-block limit concern and enabling per-card `replace_original` on button click

### vw_low_margin_products

Replaces the WC-era view. Sources:

| Table | Role |
|---|---|
| `tl_shopify_products_mirror p` | supplier name, cost, status, Shopify GIDs |
| `tl_shopify_variants_mirror v` | live price, variant GID, SKU |
| `tl_feed_leader_raw l` | RRP (joined via `p.supplier_product_id = l.stock_code`) |

**Filters:** `TRIM(p.supplier_name) = 'Leader'`, `p.status = 'ACTIVE'`, `v.price > 0`, `p.supplier_cost_inc_gst > 0`, margin < 15%

**Alert levels:**
- `CRITICAL` — `supplier_cost_inc_gst > v.price` (selling below cost)
- `WARNING` — margin < 10%

### Slack message format

Each run sends **N+1 separate messages** to `#price-watchdog`:

**Message 1 — Summary:**
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

### "Set to RRP" button

- `action_id: set_price_to_rrp`
- `value`: compact JSON `{"vGid":"...","pGid":"...","rrp":"X.XX","sku":"..."}`
- Routed through `TL_Slack_Interaction_Handler_Shopify` (`hikIeVV081e76pEv`) — Switch case 10
- On success: clicked card replaced with `:white_check_mark: Price updated to $X.XX (RRP) for *SKU* by @user`
- Uses `replace_original: true` — replaces only the clicked product card (works cleanly because each product is its own message)

**Button handler branch (in `TL_Slack_Interaction_Handler_Shopify`):**
```
Set Price Get Token   → POST Shopify OAuth client_credentials
  → Update Price in Shopify   → GraphQL productVariantsBulkUpdate
  → Update Variants Mirror    → Postgres UPDATE tl_shopify_variants_mirror SET price = rrp
  → Confirm Price Updated     → POST response_url (replace_original: true)
```

---

## WI-B — TL_Queue_Reviewer_Shopify

**n8n ID:** `20i0wyIaAnu9h2Wh`  
**Status:** Active since 2026-05-31 — no rewrite needed.

The Queue Reviewer reads from `tl_onboarding_queue` (not WC-specific). It was already cloned and active as part of Phase 10/11 clean-up work. The HITL Approve/Reject buttons route through `TL_Slack_Interaction_Handler_Shopify`.

---

## WI-C — One-off Bulk RRP Update

**Script:** `/Users/regan.mcgregor/Obsidian/TechLoop/_scratch/bulk-rrp-update.mjs`

Before the daily watchdog was active, ~333 Leader products were already selling below or near cost because the WC-era prices were imported without RRP corrections. This one-off CLI script bulk-corrected them.

**What it does:**
1. Queries `vw_low_margin_products WHERE leader_rrp IS NOT NULL AND leader_rrp > 0`
2. For each row: POSTs `productVariantsBulkUpdate` GraphQL mutation to set `price = leader_rrp`
3. PATCHes `tl_shopify_variants_mirror.price` to match
4. Rate-limited: 500ms between Shopify calls

**Run:**
```bash
env $(grep -v '^#' /Users/regan.mcgregor/Obsidian/TechLoop/.env | xargs) node /Users/regan.mcgregor/Obsidian/TechLoop/_scratch/bulk-rrp-update.mjs
```

**Result (2026-06-18):** 333 updated, 0 failed.

**Dry-run:** pass `--dry-run` to preview without writing.

---

## Credentials

| Credential | n8n ID | Used for |
|---|---|---|
| Supabase Postgres | `BSoGuZ9BOv4OWqXf` | `vw_low_margin_products` query + mirror update |
| TechLoop Automation (Slack) | `DvtYgxYgI1p5Q3QX` | Send alert to `#price-watchdog` |
| Shopify Client Credentials | `4Kf8ZBknzZl6bvvi` | Token fetch for GraphQL price mutation |

---

## Gotchas

- **Slack 50-block limit**: The original single-message design with 15 products × 3 blocks = 48+ blocks hit the limit. Per-product messages (2 blocks each) solve this entirely.
- **`replace_original` + combined message**: When all products were in one message, `replace_original: true` on button click wiped all other product cards. Per-product messages mean each card is independently replaceable.
- **`patchNodeField` with `$` in values**: The `Confirm Price Updated` node value contains `$` characters — use `updateNode` with full parameters, not `patchNodeField` (which mangles `$`).
- **`ACTIVE` uppercase**: `vw_low_margin_products` filter uses `p.status = 'ACTIVE'` (uppercase) to match `tl_shopify_products_mirror`.
- **`TRIM(supplier_name)`**: Some rows have trailing spaces in `supplier_name` — always `TRIM()` in the filter.
