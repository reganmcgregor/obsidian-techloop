> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 2: WooCommerce Mirroring - Implementation Complete

**Status:** Active - Running Every 4 Hours
**n8n Workflow ID:** `hEUnN85kXJZEG3qZ`
**Workflow Name:** `TL_Mirror_WooCommerce`
**Schedule:** Every 4 hours at :15 (cron: `15 */4 * * *`)
**Created:** 2026-01-13

---

## Overview

This workflow mirrors WooCommerce products into the Supabase `tl_wc_products_mirror` table with intelligent incremental syncing. On the first run, it fetches all products. On subsequent runs, it only fetches products modified since the last sync (using `modified_after` parameter).

**Key Features:**
- Automatic first-run detection (full sync vs. incremental)
- Extracts ACF fields from WooCommerce meta_data
- Batch upserts (50 products per batch) for efficiency
- Complete audit logging to `tl_workflow_executions`
- Email report with sync summary
- Email prefix for easy filtering: `[TechLoop Inventory]`

---

## Workflow Architecture

### 13 Nodes in Total

1. **Schedule Trigger** - Cron trigger (every 4 hours at :15)
2. **Start Execution Log** - Captures execution metadata
3. **Calculate Modified After** - Calculates timestamp for incremental sync (NOW - 4.25 hours)
4. **Check If First Run** - Queries database to check if table is empty
5. **Determine Sync Mode** - Decides full vs. incremental sync
6. **Fetch WC Products** - Calls WooCommerce REST API with optional `modified_after` filter
7. **Extract ACF Fields** - Parses meta_data array to extract ACF supplier fields
8. **Batch Products** - Groups products into batches of 50
9. **Upsert to Supabase** - Batch upserts using PostgreSQL INSERT...ON CONFLICT
10. **Aggregate Results** - Calculates totals and duration (fixed to count items correctly)
11. **Log Execution to Database** - Records execution in `tl_workflow_executions`
12. **Prepare Email Report** - Formats HTML email with stats and prefix
13. **Send Email Report** - Sends email to regan@techloop.com.au

---

## Data Extraction

### ACF Fields Extracted

From WooCommerce `meta_data` array, the workflow extracts:

- `supplier_name` → `acf_supplier_name`
- `supplier_product_id` → `acf_supplier_product_id`
- `supplier_dbp` → `acf_supplier_dbp`
- `supplier_mpn` → `acf_supplier_mpn`
- `supplier_barcode` → `acf_supplier_barcode`

### Additional Fields Captured

- Standard WooCommerce fields (sku, name, status, stock_quantity, prices)
- `_wc_cog_cost` (Cost of Goods from WooCommerce Cost of Goods plugin)
- Full `raw_meta_data` as JSONB (for future-proofing)
- Categories and tags as JSONB arrays
- Date created/modified timestamps

---

## Incremental Sync Logic

### First Run Detection

```sql
SELECT COUNT(*) as row_count FROM tl_wc_products_mirror
```

If `row_count = 0`, it's the first run → **Full Sync**
If `row_count > 0`, it's a subsequent run → **Incremental Sync**

### Modified After Calculation

For incremental syncs:
```javascript
const hoursAgo = 4.25; // 4 hours + 15 minutes overlap
const modifiedAfter = new Date(Date.now() - (hoursAgo * 60 * 60 * 1000));
```

The 15-minute overlap ensures no products are missed between sync intervals.

### WooCommerce API Call

**Full Sync:**
```
GET /wp-json/wc/v3/products?per_page=100
```

**Incremental Sync:**
```
GET /wp-json/wc/v3/products?per_page=100&modified_after=2026-01-13T00:00:00Z
```

---

## Database Operations

### Upsert Query

The workflow uses a PostgreSQL `INSERT...ON CONFLICT` upsert that:

1. Inserts new products
2. Updates existing products (matched by `wc_id`)
3. Increments `sync_count` on updates
4. Updates `updated_at` timestamp automatically

**Key Features:**
- Handles NULL values gracefully
- Parses JSONB for `raw_meta_data`, `categories`, `tags`
- Converts string prices to numeric
- Returns `wc_id` of all affected rows

### Batch Processing

Products are processed in batches of **50** to:
- Reduce database round trips
- Stay within PostgreSQL parameter limits (max ~65k)
- Provide progress visibility in logs

---

## Email Report

Sent to: `regan@techloop.com.au`
From: `noreply@techloop.com.au`

### Report Contents

**HTML formatted email with:**
- Sync mode (Full Sync or Incremental Sync)
- Products processed count
- Products updated count
- Execution duration (seconds)
- Start and completion timestamps (Australia/Sydney timezone)
- Execution ID for reference

**Subject Line Examples:**
- `[TechLoop Inventory] ✅ WooCommerce Mirror Sync Complete - 485 products (full)`
- `[TechLoop Inventory] ✅ WooCommerce Mirror Sync Complete - 12 products (incremental)`

---

## Recent Fixes

### Email Aggregation Bug (2026-01-23)

**Issue:** Email showed 0 products processed due to incorrect array check in Aggregate Results node.

**Root Cause:** Code checked `if (Array.isArray(item.json))` but `item.json` contained objects like `{wc_id: 123}`, not arrays.

**Fix:** Changed to count items directly:
```javascript
let totalProcessed = items.length; // Each item represents one processed product
let totalUpdated = items.length; // All are updates for incremental sync
```

### Email Prefix Addition (2026-01-23)

**Enhancement:** Added `[TechLoop Inventory]` prefix to all email subjects for easier filtering in email client.

---

## Error Handling

### Retry Configuration

**Node: Check If First Run**
- `retryOnFail: true`
- `maxTries: 3`
- Handles temporary database connection issues

**Node: Upsert to Supabase**
- `retryOnFail: true`
- `maxTries: 3`
- `continueOnFail: true` (logs partial failures)

**Node: Log Execution to Database**
- `retryOnFail: true`
- `maxTries: 2`
- Ensures execution is logged even on network hiccups

---

## Performance Estimates

### First Run (Full Sync)

Assuming 500 products in WooCommerce:
- API calls: ~5 requests (100 products per page)
- Processing time: ~30-60 seconds
- Database batches: 10 batches (50 products each)
- Total duration: ~2-3 minutes

### Subsequent Runs (Incremental)

Assuming 10 modified products:
- API calls: 1 request
- Processing time: ~5-10 seconds
- Database batches: 1 batch
- Total duration: ~15-30 seconds

**Efficiency Gain:** ~95% reduction in API calls and processing time

---

## Credentials Used

### WooCommerce API
- **Credential ID:** `42TWte8V6ueMoIja`
- **Credential Name:** WooCommerce account
- **Access:** Read-only access to products

### Supabase PostgreSQL
- **Credential ID:** `supabase`
- **Credential Name:** Supabase PostgreSQL
- **Database:** TechLoop Inventory
- **Tables:** `tl_wc_products_mirror`, `tl_workflow_executions`

### SMTP Email
- **Credential ID:** `vkCbG62NopBnWPHl`
- **Credential Name:** Elastic Email
- **From:** noreply@techloop.com.au
- **To:** regan@techloop.com.au

---

## Monitoring and Validation

### Database Queries

**Check last sync execution:**
```sql
SELECT *
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Mirror_WooCommerce'
ORDER BY started_at DESC
LIMIT 1;
```

**Check product count in mirror:**
```sql
SELECT COUNT(*) as total_products,
       COUNT(DISTINCT acf_supplier_name) as supplier_count,
       MAX(updated_at) as last_updated
FROM tl_wc_products_mirror;
```

**Check products with Leader supplier:**
```sql
SELECT COUNT(*) as leader_products
FROM tl_wc_products_mirror
WHERE acf_supplier_name = 'Leader';
```

### Success Criteria

- ✅ Workflow executes every 4 hours at :15 past the hour
- ✅ First run syncs all WooCommerce products (full sync)
- ✅ Subsequent runs only fetch modified products (incremental)
- ✅ All ACF fields extracted correctly
- ✅ Execution logged to `tl_workflow_executions`
- ✅ Email report received after each sync with correct counts
- ✅ Email subjects prefixed with `[TechLoop Inventory]`
- ✅ No duplicate products in mirror table

---

## Integration with Other Workflows

### Upstream Dependencies
- **None** - This workflow is independent

### Downstream Consumers
- **Workflow 3: Inventory Syncer** - Will join `tl_wc_products_mirror` with `tl_feed_leader_raw` to detect differences
- **Workflow 4: Price Watchdog** - Will query `tl_wc_products_mirror` for margin analysis

---

## Related Documentation

- [[Master Note - Inventory Automation]] - Project overview
- [[Implementation/Phase 2 - Ingestion]] - Phase documentation
- [[../../3-Resources/Infrastructure/Database Schema - TechLoop Inventory|Database Schema]] - Table definitions
- Plan file: `/Users/regan.mcgregor/.claude/plans/robust-forging-blossom.md`

---

## Technical Specifications

### Workflow Settings
- **Execution Order:** v1 (sequential)
- **Save Error Executions:** All
- **Save Success Executions:** All
- **Save Manual Executions:** Yes
- **Timezone:** UTC (converted to Australia/Sydney in reports)

### Node Count by Type
- **Trigger nodes:** 1 (Cron)
- **Code nodes:** 5 (JavaScript transformations)
- **Database nodes:** 2 (PostgreSQL queries)
- **API nodes:** 1 (WooCommerce)
- **Email nodes:** 1 (SMTP)

---

**Created by:** opencode (continuing from Claude Code)
**Date:** 2026-01-23
**Status:** ✅ Active and Fixed</content>
<parameter name="filePath">/Users/regan.mcgregor/Library/Mobile Documents/iCloud~md~obsidian/Documents/TechLoop/1-Projects/TechLoop Inventory Automation/Implementation/Workflow 3 - Inventory Syncer - IMPLEMENTED.md
