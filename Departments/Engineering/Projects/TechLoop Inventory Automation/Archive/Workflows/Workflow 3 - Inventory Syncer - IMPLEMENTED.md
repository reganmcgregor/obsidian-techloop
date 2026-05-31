> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 3: Inventory Syncer - Implementation Complete

**Status:** Created - Requires Manual Activation
**n8n Workflow ID:** `onR26JqfxIVpwWqt`
**Workflow Name:** `TL_Inventory_Syncer`
**Schedule:** Every 3 hours at :30 (cron: `30 */3 * * *`)
**Created:** 2026-01-24

---

## Overview

This is the core "Diff Engine" of the inventory automation system. It compares the raw Leader feed data (`tl_feed_leader_raw`) with the current WooCommerce state (`tl_wc_products_mirror`) to identify and execute necessary updates.

**Key Logic:**
- **Stock Only:** Update stock quantity and status.
- **Cost Only:** Update `_wc_cog_cost` and `acf_supplier_dbp`.
- **Both:** Update all of the above.
- **Discontinued:** Set stock to 0 and status to `outofstock` if missing from feed or stale (>2 hours).

---

## Workflow Architecture

### 13 Nodes

1. **Schedule Trigger** - Every 3 hours at :30
2. **Capture Start Time** - Logging start
3. **Calculate Diff** - Complex SQL JOIN query
4. **Route Changes** - Switch node (Stock, Cost, Both, Discontinued)
5. **Update Stock** - WooCommerce API
6. **Update Cost** - WooCommerce API
7. **Update Both** - WooCommerce API
8. **Discontinue** - WooCommerce API
9. **Merge Changes** - Merges all update branches
10. **Aggregate Results** - Calculates statistics
11. **Log Execution** - DB logging
12. **Prepare Email Report** - HTML generation
13. **Send Email Report** - SMTP

---

## The Diff Query

```sql
SELECT
  w.wc_id,
  w.sku,
  w.name,
  w.acf_supplier_product_id,
  l.stock_code,
  l.availability_total as leader_stock,
  l.dbp as leader_cost,
  w.stock_quantity as wc_stock,
  w.wc_cog_cost as wc_cost,
  CASE
    WHEN l.stock_code IS NULL THEN 'discontinued'
    WHEN l.last_seen_at < NOW() - INTERVAL '2 hours' THEN 'discontinued'
    WHEN l.availability_total != w.stock_quantity AND l.dbp != w.wc_cog_cost THEN 'stock_and_cost'
    WHEN l.availability_total != w.stock_quantity THEN 'stock_only'
    WHEN l.dbp != w.wc_cog_cost THEN 'cost_only'
    ELSE 'no_change'
  END AS change_type
FROM tl_wc_products_mirror w
LEFT JOIN tl_feed_leader_raw l ON w.acf_supplier_product_id = l.stock_code
WHERE w.acf_supplier_name = 'Leader'
  AND (
    l.stock_code IS NULL OR
    l.last_seen_at < NOW() - INTERVAL '2 hours' OR
    l.availability_total != w.stock_quantity OR
    l.dbp != w.wc_cog_cost
  )
LIMIT 1000;
```

---

## Manual Activation Required

The workflow has been created but **must be activated manually** in n8n.

1. Open n8n: https://n8n.reganmcgregor.com.au
2. Find **TL_Inventory_Syncer**
3. Open it and check for any validation warnings (orange triangles)
4. Click **Activate** (top right)

---

## Validation Steps

1. Run manually once (click "Execute Workflow").
2. Check `tl_workflow_executions` table for the log entry.
3. Check email for the report.
4. Verify a specific product update in WooCommerce.

---

**Created by:** Gemini CLI
**Date:** 2026-01-24
