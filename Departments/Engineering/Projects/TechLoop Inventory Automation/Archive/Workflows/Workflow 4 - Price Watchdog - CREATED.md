> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 4: Price Watchdog - Implementation Complete

**Status:** Created - Requires Manual Activation
**n8n Workflow ID:** `yExNIbiIGVFOoJ4N`
**Workflow Name:** `TL_Price_Watchdog`
**Schedule:** Daily at 6:00 AM (cron: `0 6 * * *`)
**Created:** 2026-01-24

---

## Overview

This workflow acts as a safety net, monitoring the database for products selling below cost or with dangerously low margins. It uses the `vw_low_margin_products` view which joins real-time mirror data with supplier feed data.

**Key Features:**
- **Daily Scan:** Checks all active products every morning.
- **Smart Filtering:** Only sends emails if alerts are found.
- **Rich Reporting:** HTML email with clickable links to WooCommerce edit pages.
- **Categorization:** Distinguishes between CRITICAL (selling below cost) and WARNING (margin < 10%).

---

## Workflow Architecture

### 6 Nodes

1. **Schedule Trigger** - Daily at 6:00 AM
2. **Capture Start Time** - Logging (simplified)
3. **Get Low Margin Products** - SQL Query from `vw_low_margin_products`
4. **Check for Alerts** - IF node to stop execution if 0 results
5. **Prepare Email** - JS Code to format HTML table
6. **Send Email** - SMTP

---

## The Query

```sql
SELECT * FROM vw_low_margin_products;
```

This view handles all the complexity:
- Joins `tl_wc_products_mirror` and `tl_feed_leader_raw`
- Calculates `margin_percent`
- Determines `alert_level` (CRITICAL vs WARNING)
- Filters for `status = 'publish'` only

---

## Manual Activation Required

The workflow has been created but **must be activated manually** in n8n.

1. Open n8n: https://n8n.reganmcgregor.com.au
2. Find **TL_Price_Watchdog**
3. Open it and check for any validation warnings.
4. Click **Activate** (top right)

---

## Validation Steps

1. Run manually once (click "Execute Workflow").
2. If you have products with low margins, you should receive an email.
3. If no alerts exist, the workflow should stop at the "Check for Alerts" node (false branch).

---

**Created by:** Gemini CLI
**Date:** 2026-01-24
