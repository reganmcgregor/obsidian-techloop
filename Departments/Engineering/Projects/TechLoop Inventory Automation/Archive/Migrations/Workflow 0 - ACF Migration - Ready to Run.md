> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 0: ACF Migration - Ready to Execute

**Status:** ✅ Created in n8n | ⚠️ Requires Configuration Before Running
**Workflow ID:** ABT5WhqLikVLvbBc
**n8n URL:** https://n8n.reganmcgregor.com.au/workflow/ABT5WhqLikVLvbBc

---

## What Was Created

The one-time ACF migration workflow has been successfully created in n8n with 11 nodes:

1. **Manual Trigger** - Manual execution only (no schedule)
2. **Fetch Leader XML Feed** - Downloads complete Leader product catalog
3. **Parse Leader XML** - Converts XML to JavaScript objects
4. **Create Leader Lookup** - Indexes Leader products by ManufacturerSKU
5. **Fetch WC Products** - Gets all published WooCommerce products
6. **Match WC to Leader** - Matches WC SKU to Leader ManufacturerSKU
7. **Batch Products (25 each)** - Splits matched products into batches
8. **Prepare Batch Request** - Prepares WooCommerce batch API payload
9. **WC Batch Update** - Executes batch update via WC REST API
10. **Wait (Rate Limit)** - 2 second delay between batches
11. **Generate Report** - Creates migration summary with match statistics

---

## Configuration Required

Before running this workflow, you need to configure credentials in n8n:

### 1. WooCommerce API Credentials

**Node:** "Fetch WC Products"
**Credential Type:** WooCommerce API
**Credential Name:** `TechLoop WooCommerce`

**Values needed:**
- **URL:** https://www.techloop.com.au
- **Consumer Key:** (from WooCommerce > Settings > Advanced > REST API)
- **Consumer Secret:** (from WooCommerce > Settings > Advanced > REST API)

**How to create WooCommerce API credentials:**
1. Log into WooCommerce admin
2. Go to WooCommerce > Settings > Advanced > REST API
3. Click "Add Key"
4. Description: "n8n Inventory Automation"
5. User: (select admin user)
6. Permissions: Read/Write
7. Generate API Key
8. Copy Consumer Key and Consumer Secret

### 2. HTTP Basic Auth (for Batch Update)

**Node:** "WC Batch Update"
**Credential Type:** HTTP Basic Auth
**Credential Name:** `TechLoop WC Basic Auth`

**Values needed:**
- **Username:** (same as WooCommerce Consumer Key)
- **Password:** (same as WooCommerce Consumer Secret)

---

## Pre-Flight Checklist

Before executing the migration, complete this checklist:

- [ ] **Backup WooCommerce Database** - Critical! Always backup before mass updates
- [ ] **Configure WooCommerce API Credentials** in n8n (see above)
- [ ] **Configure HTTP Basic Auth Credentials** in n8n (see above)
- [ ] **Test Credentials** - Run a test to fetch 1-2 products first
- [ ] **Review Expected Product Count** - How many products do you expect in WooCommerce?
- [ ] **Check Leader Feed Access** - Verify the Leader XML feed URL is accessible
- [ ] **Review Match Logic** - Workflow matches WC `sku` to Leader `ManufacturerSKU`

---

## How the Matching Works

**Matching Logic:**
```
WooCommerce Product SKU = Leader ManufacturerSKU
```

**Example:**
- WooCommerce Product: SKU = "EC415B"
- Leader Product: ManufacturerSKU = "EC415B"
- ✅ MATCH → ACF fields will be populated

**What gets populated:**
```json
{
  "supplier_name": "Leader",
  "supplier_product_id": "HXSI-EC415B" (Leader StockCode),
  "supplier_dbp": "13.50" (Leader DBP),
  "supplier_mpn": "EC415B" (Leader ManufacturerSKU),
  "supplier_barcode": "9350414002765" (Leader BarCode)
}
```

---

## Expected Results

After running the migration, you should see:

**Best Case Scenario (90-100% match rate):**
- Most products matched successfully
- ACF fields populated for all matched products
- Small list of unmatched products to review manually
- Ready to start automated sync workflows

**Realistic Scenario (70-90% match rate):**
- Majority of products matched
- Some products need manual review (SKU mismatches, missing SKUs)
- Most products ready to sync

**What causes mismatches:**
1. **WooCommerce SKU ≠ Leader ManufacturerSKU** - SKU doesn't match
2. **Missing SKU** - WooCommerce product has no SKU
3. **Product not in Leader catalog** - Product from different supplier
4. **Typos in SKUs** - "EC415B" vs "EC415b" (case sensitivity)

---

## How to Run the Migration

### Step 1: Open Workflow in n8n
1. Navigate to https://n8n.reganmcgregor.com.au
2. Open workflow: "TL_Migration_Populate_ACF_Fields"
3. Verify all nodes are connected properly

### Step 2: Configure Credentials
1. Click on "Fetch WC Products" node
2. Add/Select WooCommerce API credentials
3. Click on "WC Batch Update" node
4. Add/Select HTTP Basic Auth credentials

### Step 3: Test with Single Product (Optional but Recommended)
Before running full migration, test with 1 product:

1. Modify "Fetch WC Products" node:
   - Add parameter: `limit: 1` or `per_page: 1`
2. Click "Execute Node" to test
3. Verify the output shows correct product data
4. Check if match is found in "Match WC to Leader" node
5. If successful, revert to fetch all products

### Step 4: Execute Full Migration
1. Click "Execute Workflow" button (top right)
2. Monitor execution in real-time
3. Watch for any errors (they'll show in red)
4. Wait for completion (may take 5-15 minutes depending on product count)

### Step 5: Review Migration Report
After completion:
1. Click on "Generate Report" node
2. Review the JSON output:
   - `summary.matched_and_updated` - How many products updated
   - `summary.unmatched` - How many products NOT matched
   - `summary.match_rate` - Overall success percentage
   - `unmatched_products` - List of products that need manual review

---

## Post-Migration Steps

### 1. Verify in WooCommerce Admin
Spot check a few products in WooCommerce:
1. Edit any product
2. Scroll down to "Advanced Custom Fields" section
3. Verify you see:
   - Supplier Name: "Leader"
   - Supplier Product ID: (should have value)
   - Supplier DBP: (should have value)
   - Supplier MPN: (should have value)
   - Supplier Barcode: (should have value)

### 2. Handle Unmatched Products
Review the unmatched products list:
- **No SKU:** Add SKU in WooCommerce, re-run migration
- **SKU Mismatch:** Correct SKU to match Leader ManufacturerSKU
- **Not in Leader Catalog:** If from different supplier, populate ACF manually or exclude from sync

### 3. Run Workflow 2 (WooCommerce Mirror)
Once ACF fields are populated:
1. Manually trigger Workflow 2: "TL_WooCommerce_Mirror"
2. This will populate the `tl_wc_products_mirror` table in Supabase
3. Verify data with this query:

```sql
SELECT
    COUNT(*) as total_products,
    COUNT(acf_supplier_product_id) as with_supplier_id,
    ROUND(
        (COUNT(acf_supplier_product_id)::numeric / COUNT(*) * 100),
        2
    ) as population_rate
FROM tl_wc_products_mirror
WHERE status = 'publish';
```

Expected: ~80-100% population rate

### 4. Test Sync Detection
Run this query to see if products are ready to sync:

```sql
SELECT
    COUNT(*) as products_ready_to_sync
FROM tl_wc_products_mirror
WHERE acf_supplier_name = 'Leader'
  AND acf_supplier_product_id IS NOT NULL;
```

This should match your matched product count.

---

## Troubleshooting

### Issue: "Workflow failed to execute"
**Possible causes:**
- Missing credentials
- Leader XML feed inaccessible
- WooCommerce API rate limit exceeded

**Solution:**
1. Check credentials are configured
2. Test Leader feed URL in browser
3. Check n8n execution logs for specific error

### Issue: "No products matched"
**Possible causes:**
- SKU format mismatch (case sensitivity)
- Leader feed parsed incorrectly
- WooCommerce SKUs don't match Leader ManufacturerSKUs

**Solution:**
1. Check "Create Leader Lookup" node output - verify products indexed
2. Check "Fetch WC Products" node output - verify SKUs present
3. Manually compare a few WC SKUs to Leader ManufacturerSKUs
4. May need to adjust matching logic if SKU format differs

### Issue: "WooCommerce batch update failed"
**Possible causes:**
- Invalid API credentials
- Rate limiting
- Large batch size

**Solution:**
1. Verify Basic Auth credentials match WooCommerce API keys
2. Reduce batch size from 25 to 10 in "Batch Products" node
3. Increase wait time from 2s to 5s in "Wait (Rate Limit)" node

### Issue: "Migration runs but ACF fields not populated"
**Possible causes:**
- ACF fields don't exist in WooCommerce
- Different field names
- WooCommerce caching

**Solution:**
1. Verify ACF fields exist: WooCommerce > Custom Fields
2. Check field names match exactly (case sensitive)
3. Clear WooCommerce cache
4. Re-run Workflow 2 to re-mirror

---

## Migration Report Example

Expected output from "Generate Report" node:

```json
{
  "migration_date": "2026-01-12T12:30:00.000Z",
  "workflow": "TL_Migration_Populate_ACF_Fields",
  "summary": {
    "total_wc_products": 1250,
    "matched_and_updated": 1180,
    "unmatched": 70,
    "match_rate": "94.40%"
  },
  "acf_fields_populated": [
    "supplier_name = \"Leader\"",
    "supplier_product_id = Leader StockCode",
    "supplier_dbp = Leader DBP",
    "supplier_mpn = Leader ManufacturerSKU",
    "supplier_barcode = Leader BarCode"
  ],
  "unmatched_products": [
    {
      "sku": "CUSTOM-001",
      "name": "Custom Product",
      "wc_id": 5021,
      "reason": "No matching Leader product (SKU not found in Leader feed)"
    },
    ...
  ],
  "recommendation": "Review 70 unmatched products - they may need manual ACF field population or SKU correction"
}
```

---

## Safety Features

This workflow includes several safety features:

1. **Manual Trigger Only** - Cannot run accidentally on a schedule
2. **Batch Processing** - Updates 25 products at a time to avoid overwhelming WC
3. **Rate Limiting** - 2 second delay between batches
4. **Idempotent** - Safe to re-run (will overwrite existing ACF values)
5. **Detailed Logging** - Each node logs progress to console
6. **Complete Reporting** - Full list of matched and unmatched products

---

## After Successful Migration

Once migration is complete and verified:

1. ✅ **Disable this workflow** - It's one-time only, prevent accidental re-runs
2. ✅ **Archive execution logs** - Keep the migration report for records
3. ✅ **Update Master Note** - Mark Phase 1.5 complete
4. ✅ **Proceed to Phase 2** - Build Workflow 1 (Leader Ingest) and Workflow 2 (WC Mirror)
5. ✅ **Test automated sync** - Workflow 3 (Inventory Syncer) will now have ACF data to work with

---

## Questions Before Running?

Common questions to consider:

**Q: Will this overwrite existing ACF values?**
A: Yes, if a product already has ACF supplier fields, they will be overwritten with Leader data.

**Q: What happens to products that don't match?**
A: They're logged in the unmatched_products list but not modified. You can manually populate their ACF fields later.

**Q: Can I re-run this if I need to?**
A: Yes, it's idempotent and safe to re-run. It will re-match all products and update ACF fields again.

**Q: How long will it take?**
A: Depends on product count. Approximately 1-2 minutes per 100 products (due to batch processing and rate limiting).

**Q: Will it affect live products?**
A: It only updates ACF meta fields (backend data). Frontend display is not affected until sync workflows run.

---

## Next Steps After Migration

1. **Verify population rate** - Check Supabase after running Workflow 2
2. **Build Workflow 1** - Leader Ingest (Phase 2)
3. **Build Workflow 2** - WooCommerce Mirror (Phase 2)
4. **Build Workflow 3** - Inventory Syncer (Phase 3)
5. **Start automated syncing** - Turn on schedules

Migration is Phase 1.5 - a prerequisite for all automated sync workflows.

---

**Ready to execute?** Follow the "How to Run the Migration" steps above.
