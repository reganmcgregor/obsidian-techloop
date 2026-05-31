> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 3: Inventory Syncer - The Diff Engine

**Status:** Not Yet Built
**Planned Schedule:** Every 3 hours at :30 (cron: `30 */3 * * *`)

---

## Overview

This workflow will create the core diff engine that syncs inventory and cost from Leader supplier feed to WooCommerce products.

**Purpose:** Compare `tl_feed_leader_raw` with `tl_wc_products_mirror` to detect changes and sync them automatically.

## Planned Workflow Structure

1. **Schedule Trigger** - Every 3 hours at :30
2. **Query Differences** - JOIN query to find products that need updates
3. **Route by Change Type** - Switch node for stock vs cost vs both updates
4. **Update WooCommerce** - API calls to update stock/prices
5. **Update Mirror Table** - Mark products as synced
6. **Log Changes** - Detailed audit logging
7. **Email Report** - Summary of sync operations

## Key Logic - The Diff Query

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
    WHEN l.availability_total != w.stock_quantity THEN 'stock_only'
    WHEN l.dbp != w.wc_cog_cost THEN 'cost_only'
    WHEN l.availability_total != w.stock_quantity AND l.dbp != w.wc_cog_cost THEN 'stock_and_cost'
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
  );
```

## Dependencies

- **Workflow 1:** Leader Ingest (provides fresh supplier data)
- **Workflow 2:** WooCommerce Mirroring (provides current WC state)

## Next Steps

1. Design the diff query and routing logic
2. Implement WooCommerce update API calls
3. Add error handling and retry logic
4. Test with sample changes
5. Deploy and monitor

---

**Status:** Ready for implementation after Workflow 1 & 2 are stable.
