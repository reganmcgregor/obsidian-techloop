> [!info] Part of [[Master Note - Inventory Automation]]

# EXECUTE NOW: ACF Migration - Step-by-Step Guide

**Status:** ✅ Backup Complete | 🔐 Credentials Ready | ⏳ Ready to Execute
**Workflow:** TL_Migration_Populate_ACF_Fields
**URL:** https://n8n.reganmcgregor.com.au/workflow/ABT5WhqLikVLvbBc

---

## 🔐 Your WooCommerce API Credentials

```
Consumer Key: ck_33b6dcd7f32831d56c41816a184577709f9ee38c
Consumer Secret: cs_11c71903d2520130230f748ad6a749d3f27a233c
```

**⚠️ Security Note:** These credentials grant read/write access to your WooCommerce store. Keep them secure. After migration, you may want to regenerate them if concerned about exposure.

---

## Step 1: Configure WooCommerce API Credentials (5 minutes)

1. **Open n8n:** Navigate to https://n8n.reganmcgregor.com.au
2. **Open workflow:** Click on "TL_Migration_Populate_ACF_Fields" (or use direct link above)
3. **Find the "Fetch WC Products" node** (5th node from left)
4. **Click on the node** to open its settings

### Add WooCommerce Credential:

5. In the node settings, find the **"Credential to connect with"** dropdown
6. Click **"+ Create New"** or select existing if available
7. Fill in the credential form:
   - **Credential Name:** `TechLoop WooCommerce API`
   - **WooCommerce URL:** `https://www.techloop.com.au`
   - **Consumer Key:** `ck_33b6dcd7f32831d56c41816a184577709f9ee38c`
   - **Consumer Secret:** `cs_11c71903d2520130230f748ad6a749d3f27a233c`
   - **Include Query in URL:** ❌ (unchecked)
8. Click **"Create"** to save the credential
9. The credential should now be selected in the "Fetch WC Products" node

---

## Step 2: Configure HTTP Basic Auth (2 minutes)

10. **Find the "WC Batch Update" node** (8th node from left)
11. **Click on the node** to open its settings
12. Find **"Credential to connect with"** dropdown under Authentication
13. Click **"+ Create New"**
14. Fill in the credential form:
    - **Credential Name:** `TechLoop WC Basic Auth`
    - **Username:** `ck_33b6dcd7f32831d56c41816a184577709f9ee38c` (same as Consumer Key)
    - **Password:** `cs_11c71903d2520130230f748ad6a749d3f27a233c` (same as Consumer Secret)
15. Click **"Create"** to save
16. The credential should now be selected in the "WC Batch Update" node

---

## Step 3: Test with Single Product (RECOMMENDED - 5 minutes)

Before running the full migration, test with 1 product to verify everything works:

17. **Click on "Fetch WC Products" node**
18. In the node parameters, click **"Add Option"**
19. Select **"Limit"**
20. Set value to: `1`
21. Click **"Execute Node"** (button at top of node panel)
22. **Check the output:**
    - Should show 1 product with SKU, name, meta_data
    - If error: Check credentials are correct
23. **Click on "Match WC to Leader" node**
24. Click **"Execute Node"**
25. **Check the output:**
    - Should show `matched: 1` or `unmatched: 1`
    - Review the match/unmatch reason
26. **If successful:** Remove the Limit parameter from "Fetch WC Products" node
    - Click on the node
    - Click the X next to "Limit: 1" to remove it
27. **Save the workflow** (Ctrl+S or Cmd+S)

---

## Step 4: Execute Full Migration (10-30 minutes depending on product count)

28. **Double-check:**
    - ✅ Backup completed
    - ✅ Credentials configured
    - ✅ Test passed (if you ran Step 3)
    - ✅ "Limit" parameter removed from "Fetch WC Products"

29. **Click "Execute Workflow"** button (top right corner)
30. **Watch the execution progress:**
    - Nodes will light up green as they execute
    - You'll see items flowing through the workflow
    - Progress bar at bottom shows execution status

31. **Wait for completion:**
    - Leader feed fetch: ~30 seconds
    - XML parsing: ~10 seconds
    - WC product fetch: ~30-60 seconds (depends on product count)
    - Matching: ~5 seconds
    - Batch updates: ~1-2 minutes per 100 products (due to rate limiting)

32. **Monitor for errors:**
    - Red nodes = error occurred
    - Click on red node to see error message
    - Common errors and fixes below

---

## Step 5: Review Migration Report (5 minutes)

33. **Click on "Generate Report" node** (last node)
34. **View the JSON output** in the bottom panel
35. **Record key statistics:**

```
Expected output format:
{
  "summary": {
    "total_wc_products": ___,
    "matched_and_updated": ___,
    "unmatched": ___,
    "match_rate": "____%"
  },
  "unmatched_products": [...]
}
```

36. **Assess results:**
    - **90-100% match rate:** Excellent! Most products ready to sync
    - **70-90% match rate:** Good! Review unmatched products
    - **<70% match rate:** Review SKU matching logic, may need adjustment

---

## Step 6: Verify in WooCommerce (5 minutes)

37. **Open WooCommerce admin:** https://www.techloop.com.au/wp-admin
38. **Go to Products** > All Products
39. **Edit any random product**
40. **Scroll down to "Advanced Custom Fields" section** (or wherever ACF fields display)
41. **Verify these fields are populated:**
    - ✅ Supplier Name: "Leader"
    - ✅ Supplier Product ID: (should have a value like "HXSI-EC415B")
    - ✅ Supplier DBP: (should have a price like "13.50")
    - ✅ Supplier MPN: (should match the product SKU)
    - ✅ Supplier Barcode: (should have EAN/UPC)

42. **Spot check 3-5 more products** to confirm consistency

---

## Step 7: Handle Unmatched Products (Time varies)

43. **Review the unmatched_products list** from Step 5
44. **For each unmatched product, determine:**
    - **No SKU:** Add SKU in WooCommerce, then re-run migration OR populate ACF manually
    - **SKU doesn't match Leader:** Check if SKU typo, correct it, re-run migration
    - **Product not in Leader catalog:** If from different supplier, exclude from sync (populate ACF manually with different supplier)

45. **Options for handling unmatched:**
    - **Option A:** Fix SKUs in WooCommerce, re-run entire migration (safe, idempotent)
    - **Option B:** Manually populate ACF fields for those products
    - **Option C:** Leave unmatched (they won't sync automatically)

---

## Common Errors & Solutions

### Error: "401 Unauthorized" or "Invalid signature"
**Cause:** Wrong API credentials
**Solution:**
- Double-check Consumer Key and Secret are correct
- Ensure no extra spaces when copying
- Regenerate API keys in WooCommerce if needed

### Error: "Request timeout" on Leader feed
**Cause:** Leader feed is slow or temporarily unavailable
**Solution:**
- Wait 5 minutes, try again
- Check Leader feed URL in browser to verify it loads
- Increase timeout in "Fetch Leader XML Feed" node to 120000ms (2 minutes)

### Error: "Too many requests" or "429" on WooCommerce
**Cause:** Rate limiting triggered
**Solution:**
- Increase wait time in "Wait (Rate Limit)" node from 2s to 5s
- Reduce batch size in "Batch Products" node from 25 to 10

### Error: "Cannot read property 'json' of undefined"
**Cause:** Missing data from previous node
**Solution:**
- Check if all previous nodes executed successfully
- Verify Leader feed returned data (check "Parse Leader XML" node output)
- Verify WC products were fetched (check "Fetch WC Products" node output)

---

## Post-Migration Checklist

After successful migration:

- [ ] **Save migration report** - Export JSON from "Generate Report" node, save to file
- [ ] **Disable Workflow 0** - This is one-time only, disable to prevent accidental re-runs
- [ ] **Document match rate** - Record statistics in project notes
- [ ] **Log unmatched products** - Keep list for manual review
- [ ] **Proceed to Phase 2** - Now ready to build Workflow 1 (Leader Ingest) and Workflow 2 (WC Mirror)

---

## Next Step After Migration: Run Workflow 2

Once ACF fields are populated, you need to run Workflow 2 (WooCommerce Mirror) to populate the Supabase database:

**Note:** Workflow 2 doesn't exist yet - it's the next task in Phase 2. But when it's built, running it will:
1. Fetch all WooCommerce products (with newly populated ACF fields)
2. Store them in `tl_wc_products_mirror` table in Supabase
3. Extract ACF fields into dedicated columns for fast querying

**Then verify with this query:**
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

Expected: ~80-100% population rate (should match your migration match rate)

---

## Questions During Execution?

If you encounter issues:

1. **Check n8n execution logs:** Click on failed node, read error message
2. **Check this guide:** Common errors section above
3. **Test individual nodes:** Execute nodes one at a time to isolate issue
4. **Review credentials:** Most issues are credential-related
5. **Check Leader feed:** Verify it's accessible in browser

---

## Estimated Total Time

- Credential configuration: ~10 minutes
- Test execution: ~5 minutes
- Full migration: ~15-30 minutes (depends on product count)
- Verification: ~10 minutes
- **Total: 40-55 minutes**

---

**Ready to begin?** Start with Step 1 above. Good luck! 🚀
