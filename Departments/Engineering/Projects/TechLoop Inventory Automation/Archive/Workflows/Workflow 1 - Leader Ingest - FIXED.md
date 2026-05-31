> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 1: Leader Ingest - FIXED

**Date:** 2026-01-13
**Status:** ✅ Ready for Testing
**Workflow ID:** mUkhS0BfN7Lq6EsS

---

## Issues Identified and Fixed

### Issue 1: Supabase Insert Node Configuration Error ✅ FIXED

**Problem:**
- The Supabase Insert node was configured with `dataToSend: "defineBelow"` but had NO field mappings (`fieldsUi: {}`)
- This caused it to attempt inserting rows with ALL null values
- Error: `null value in column "stock_code" of relation "tl_feed_leader_raw" violates not-null constraint`

**Evidence:**
- Postgres error showed: "Failing row contains (null, null, null, null...)" - all 50+ columns were null
- Upstream "Upsert to Supabase" node successfully output 10 valid items with proper `stock_code` values like "COB-TN-2130"
- Data was correct, but the Supabase node wasn't mapping it

**Fix Applied:**
```javascript
// Changed from:
{
  "dataToSend": "defineBelow",
  "fieldsUi": {}  // Empty - no mappings!
}

// To:
{
  "dataToSend": "autoMapInputData"  // Auto-map all fields
}
```

**Result:** The node will now automatically map incoming JSON fields (`stock_code`, `bar_code`, `product_name`, etc.) to matching database columns.

---

### Issue 2: Field Mapping - Prices and Manufacturer SKU ✅ FIXED

**Problem:**
- XML field `DBP` → `d_b_p` via toSnakeCase(), but database expects `dbp`
- Similarly: `DBP5` → `d_b_p5`, `RRP` → `r_r_p`, `ManufacturerSKU` → `manufacturer_s_k_u`
- Code was trying to sanitize fields that didn't exist yet
- Result: Prices and manufacturer_sku columns were NULL in database

**Evidence:**
```sql
SELECT stock_code, dbp, dbp5, rrp, manufacturer_sku FROM tl_feed_leader_raw LIMIT 3;
-- All showed NULL for these columns
```

**Fix Applied:**
```javascript
// Map oddly-named fields to database column names
// DBP -> d_b_p, but DB expects 'dbp'
result.dbp = sanitizeDecimal(result.d_b_p);
result.dbp5 = sanitizeDecimal(result.d_b_p5);
result.rrp = sanitizeDecimal(result.r_r_p);
result.standard_rrp = sanitizeDecimal(result.standard_r_r_p);

// ManufacturerSKU -> manufacturer_s_k_u, but DB expects 'manufacturer_sku'
result.manufacturer_sku = result.manufacturer_s_k_u || null;
```

**Result:** Prices and manufacturer SKU will now be properly extracted and stored.

---

### Issue 3: 10-Item Test Limit (Temporary) 🧪 TESTING

**Purpose:**
- Validate fixes with small dataset before processing full 21,000 products
- Current code in Transform and Batch node:
```javascript
// **TESTING: LIMIT TO 10 ITEMS**
items = items.slice(0, 10);
```

**Next Steps:**
1. Test workflow with 10 items
2. Verify prices and manufacturer_sku are populated
3. Remove the `.slice(0, 10)` line
4. Test with full dataset

---

## Test Instructions

### 1. Manual Test in n8n (10-Item Test)

**IMPORTANT: Current workflow is limited to 10 items for testing**

1. Open workflow: https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS
2. Click **"Execute Workflow"** button (top right)
3. Wait for execution to complete (~30-60 seconds for 10 items)

### 2. Expected Results (10-Item Test)

**Success Criteria:**
- Execution status: ✅ Success
- 10 products ingested into `tl_feed_leader_raw` table
- **CRITICAL:** Prices (dbp, dbp5, rrp) should NOT be NULL
- **CRITICAL:** manufacturer_sku should NOT be NULL
- Email sent to regan@techloop.com.au with success summary
- Execution logged to `tl_workflow_executions`

**Email Report Should Show:**
- Total Products: 10
- Success: 10
- Failed: 0
- Duration: ~30-60 seconds

### 3. Verify in Supabase

```sql
-- CRITICAL: Check that prices and manufacturer_sku are populated
SELECT
  stock_code,
  product_name,
  dbp,           -- Should NOT be NULL
  dbp5,          -- Should NOT be NULL
  rrp,           -- Should NOT be NULL
  manufacturer_sku,  -- Should NOT be NULL
  availability_total,
  last_seen_at
FROM tl_feed_leader_raw
ORDER BY last_seen_at DESC
LIMIT 10;
-- Expected: Most recent 10 products with ALL fields populated

-- Check total products ingested (should be 10 after test)
SELECT COUNT(*) FROM tl_feed_leader_raw
WHERE last_seen_at > NOW() - INTERVAL '5 minutes';
-- Expected: 10

-- Verify prices are not NULL
SELECT COUNT(*) FROM tl_feed_leader_raw
WHERE last_seen_at > NOW() - INTERVAL '5 minutes'
  AND (dbp IS NULL OR manufacturer_sku IS NULL);
-- Expected: 0 (all should have values)

-- Check workflow execution log
SELECT
  workflow_name,
  status,
  total_items,
  success_count,
  fail_count,
  duration_seconds,
  started_at
FROM tl_workflow_executions
ORDER BY started_at DESC
LIMIT 1;
-- Expected: status='success', total_items=10, success_count=10, fail_count=0
```

### 4. Check Execution Logs

If the test fails, check execution logs:

```bash
# Via n8n MCP tool
n8n_executions(action: "list", workflowId: "mUkhS0BfN7Lq6EsS", limit: 1)
n8n_executions(action: "get", id: "EXECUTION_ID", mode: "error")
```

---

## Root Cause Analysis

**Why did this happen?**

The workflow was built by a parallel agent that incorrectly configured the Supabase Insert node. The agent set:
- `dataToSend: "defineBelow"` (manual field mapping)
- But provided no field mappings (`fieldsUi: {}`)

This is likely because:
1. The agent assumed n8n would auto-detect fields from the upstream data
2. The agent was unfamiliar with the Supabase node's configuration requirements
3. No validation was performed after workflow creation

**Lesson Learned:**
- Always validate Supabase node configuration after creation
- Use `autoMapInputData` for dynamic field mapping from upstream nodes
- Test with sample data before running full dataset

---

## Next Steps

### Phase 1: 10-Item Test (CURRENT)

1. ✅ **Syntax Fix Applied** - Transform and Batch node corrected (22:08 PM)
2. ✅ **Field Mapping Fixed** - Prices and manufacturer_sku mapping added
3. 🧪 **Manual Test Required** - User must test in n8n UI with 10 items
4. 🔍 **Verify Database** - Check that prices and manufacturer_sku are populated

### Phase 2: Full Dataset Test (AFTER 10-ITEM TEST SUCCEEDS)

**To enable full dataset:**
1. Open workflow: https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS
2. Click on "Transform and Batch" node
3. Find and remove this line:
   ```javascript
   items = items.slice(0, 10);
   ```
4. Save workflow
5. Execute workflow again
6. Wait ~3-5 minutes for all ~21,000 products to process

**Expected Results:**
- Total Products: ~21,000
- Success: ~21,000
- Failed: 0
- Duration: ~180-300 seconds
- All products have prices and manufacturer_sku populated

### Phase 3: Production (AFTER FULL TEST SUCCEEDS)

1. ⏰ **Automatic Runs** - Workflow will run every 2 hours at :00 (next: 23:00, 01:00, 03:00, etc.)
2. 📧 **Monitor Email Reports** - Check regan@techloop.com.au for execution summaries
3. 📊 **Verify Data** - Periodically query Supabase to confirm products are being updated
4. ✅ **Move to Next Workflow** - Activate and test Workflow 2 (WooCommerce Mirroring)

---

## Workflow Status: ✅ READY FOR 10-ITEM TEST

All blocking issues have been resolved. The workflow is now configured correctly with proper field mapping.

**Action Required:**
1. Manually execute the workflow in n8n UI
2. Verify prices (dbp, dbp5, rrp) and manufacturer_sku are populated in database
3. If test succeeds, remove the 10-item limit and test with full dataset
