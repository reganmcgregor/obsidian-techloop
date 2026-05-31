> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 2: WooCommerce Mirroring - Implementation Complete

**Status:** Created - Ready for Manual Activation
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
10. **Aggregate Results** - Calculates totals and duration
11. **Log Execution to Database** - Records execution in `tl_workflow_executions`
12. **Prepare Email Report** - Formats HTML email with stats
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
- `✅ WooCommerce Mirror Sync Complete - 485 products (full)`
- `✅ WooCommerce Mirror Sync Complete - 12 products (incremental)`

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

## Manual Activation Required

The workflow has been created but needs to be **manually activated** in the n8n UI:

### Steps to Activate

1. Open n8n: https://n8n.reganmcgregor.com.au
2. Navigate to **Workflows**
3. Find workflow: `TL_Mirror_WooCommerce`
4. Click the **Activate** toggle (top right)
5. Verify the cron schedule shows: `15 */4 * * *`

### Verify First Run

After activation, trigger a manual execution to verify:

1. Open the workflow in n8n
2. Click **Execute Workflow** (manual run)
3. Verify all nodes complete successfully
4. Check Supabase table `tl_wc_products_mirror` has data
5. Check email inbox for sync report

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
- ✅ Email report received after each sync
- ✅ No duplicate products in mirror table

---

## Troubleshooting

### Issue: No Products Synced

**Check:**
1. WooCommerce API credentials valid
2. WooCommerce REST API enabled
3. Products exist with `date_modified` within sync window
4. Check n8n execution logs for errors

### Issue: ACF Fields Empty

**Check:**
1. ACF fields exist in WooCommerce (supplier_name, supplier_product_id, etc.)
2. Field names match exactly (case-sensitive)
3. Check `raw_meta_data` in database to see what meta_data was captured

### Issue: Email Not Received

**Check:**
1. SMTP credentials valid (Elastic Email)
2. Email not in spam folder
3. Check n8n execution logs for email send errors
4. Verify `fromEmail` and `toEmail` addresses

---

## Next Steps

1. **Activate workflow** in n8n UI (see instructions above)
2. **Run manual test** to verify first run (full sync)
3. **Wait 4 hours** and verify incremental sync works
4. **Monitor email reports** for next 24 hours
5. **Proceed to Workflow 3** (Inventory Syncer - The Diff Engine)

---

## Integration with Other Workflows

### Upstream Dependencies
- **None** - This workflow is independent

### Downstream Consumers
- **Workflow 3: Inventory Syncer** - Joins `tl_wc_products_mirror` with `tl_feed_leader_raw` to detect differences
- **Workflow 4: Price Watchdog** - Queries `tl_wc_products_mirror` for margin analysis

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

**Created by:** Claude Code (Sonnet 4.5)
**Date:** 2026-01-13
**Status:** ✅ Ready for Manual Activation
