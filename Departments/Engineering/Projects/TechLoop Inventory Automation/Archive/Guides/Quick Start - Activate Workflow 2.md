> [!info] Part of [[Master Note - Inventory Automation]]

# Quick Start: Activate Workflow 2 - WooCommerce Mirroring

**Workflow:** `TL_Mirror_WooCommerce`
**Workflow ID:** `hEUnN85kXJZEG3qZ`
**Status:** Created, awaiting manual activation

---

## Activation Steps

### 1. Open n8n

Visit: https://n8n.reganmcgregor.com.au

### 2. Find the Workflow

- Navigate to **Workflows** in left sidebar
- Find: **TL_Mirror_WooCommerce**
- Click to open

### 3. Activate

- Click the **Activate** toggle in top right corner
- Toggle should turn green/blue
- Verify schedule shows: `15 */4 * * *` (every 4 hours at :15)

---

## Test Run (Manual Execution)

### Before First Automated Run

1. Click **Execute Workflow** button (top right)
2. Watch nodes execute in sequence (left to right)
3. Check each node has green checkmark ✓
4. Review final node output for summary

### Expected Results

**First Run (Full Sync):**
- All WooCommerce products fetched
- ~500+ products processed (depends on your catalog)
- Duration: 2-3 minutes
- Email report sent to regan@techloop.com.au

**Subsequent Runs (Incremental):**
- Only modified products fetched
- ~10-50 products typically (much faster)
- Duration: 15-30 seconds
- Email report sent

---

## Verify in Database

### Check Products Mirrored

```sql
SELECT COUNT(*) as total_products,
       COUNT(CASE WHEN acf_supplier_name = 'Leader' THEN 1 END) as leader_products,
       MAX(updated_at) as last_sync
FROM tl_wc_products_mirror;
```

### Check Execution Log

```sql
SELECT workflow_name, status, records_processed, duration_seconds, started_at
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Mirror_WooCommerce'
ORDER BY started_at DESC
LIMIT 5;
```

### Sample Product with ACF Fields

```sql
SELECT wc_id, sku, name,
       acf_supplier_name,
       acf_supplier_product_id,
       acf_supplier_dbp,
       stock_quantity
FROM tl_wc_products_mirror
WHERE acf_supplier_name = 'Leader'
LIMIT 5;
```

---

## Check Your Email

Look for email from: `noreply@techloop.com.au`
Subject: `✅ WooCommerce Mirror Sync Complete - X products (full/incremental)`

The email will show:
- Sync mode (full or incremental)
- Number of products processed
- Execution duration
- Start/completion timestamps

---

## Troubleshooting

### Workflow Won't Activate

- Check WooCommerce API credentials are valid
- Check Supabase credentials are valid
- Check SMTP credentials are valid

### No Products Found

- Verify WooCommerce REST API is enabled
- Check WooCommerce has products published
- Check API endpoint URL is correct

### ACF Fields Not Extracted

- Verify ACF plugin installed in WooCommerce
- Check field names match exactly:
  - `supplier_name`
  - `supplier_product_id`
  - `supplier_dbp`
  - `supplier_mpn`
  - `supplier_barcode`

---

## Schedule Confirmation

Once activated, workflow will run:
- **12:15 AM** (00:15)
- **4:15 AM** (04:15)
- **8:15 AM** (08:15)
- **12:15 PM** (12:15)
- **4:15 PM** (16:15)
- **8:15 PM** (20:15)

Times in Australia/Sydney timezone.

---

## Next Steps After Activation

1. ✅ Activate workflow
2. ✅ Run manual test
3. ✅ Verify database has products
4. ✅ Check email received
5. ⏳ Wait 4 hours for first automated run
6. ⏳ Verify incremental sync works (check execution log)
7. → Proceed to Workflow 3: Inventory Syncer

---

**Ready to activate?** Open n8n now: https://n8n.reganmcgregor.com.au
