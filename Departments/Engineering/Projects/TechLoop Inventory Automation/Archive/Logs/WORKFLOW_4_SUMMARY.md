> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 4: Price Watchdog - Quick Summary

**Status:** ✅ Created Successfully (Needs Activation)
**Date:** 2026-01-13
**Workflow ID:** `Zj4oIIcN5Wm8eF26`

---

## What Was Built

A daily automated workflow that monitors product pricing margins and sends email alerts when products need attention.

### Key Features

1. **Daily Schedule:** Runs automatically at 6:00 AM
2. **Smart Alerts:** Only sends email if problems exist
3. **Two Alert Levels:**
   - **CRITICAL:** Selling below cost (immediate action needed)
   - **WARNING:** Margin below 10% (review recommended)
4. **Beautiful Email Reports:** HTML formatted with colour-coding
5. **Full Audit Trail:** Logs all executions to database

---

## How It Works

```
Every day at 6:00 AM:
  ↓
Query Supabase for low-margin products
  ↓
Check if any alerts exist
  ├─ YES → Build HTML email → Send to regan@techloop.com.au → Log success
  └─ NO → Log "no alerts" and exit quietly
```

---

## Email Report Contents

When alerts exist, you'll receive an email with:

1. **Summary Cards:**
   - CRITICAL alert count (products selling below cost)
   - WARNING alert count (products with margin < 10%)
   - Total potential loss (dollars being lost on CRITICAL items)

2. **Detailed Table:**
   - SKU, Product Name, Price, Cost, Margin $, Margin %
   - Colour-coded rows (red = CRITICAL, yellow = WARNING)
   - Sorted by severity (CRITICAL first)

3. **Recommended Actions:**
   - Immediate steps for CRITICAL items
   - Review guidance for WARNING items
   - Data verification suggestions

---

## ⚠️ IMPORTANT: Next Step Required

**The workflow has been created but is NOT ACTIVE yet.**

### To Activate:

1. Go to: https://n8n.reganmcgregor.com.au
2. Find workflow: `TL_Price_Watchdog`
3. Toggle the switch to activate

**After activation:**
- First run will be tomorrow at 6:00 AM
- You'll receive email if any products have low margins
- No email = all margins are healthy

---

## Testing Before First Run

Optional: You can test immediately without waiting for 6:00 AM:

1. Open workflow in n8n UI
2. Click "Test workflow" button
3. Check your email for results
4. Verify execution logged to database

---

## What to Expect

### If You Get An Email
- **CRITICAL items:** Review pricing immediately - you're losing money
- **WARNING items:** Plan price increases to improve margins

### If You Don't Get An Email
- Good news! All your margins are healthy (above 10%)
- Workflow still ran successfully (check database logs to confirm)

---

## Database Logging

Every execution is logged to `tl_workflow_executions` table:

```sql
-- View recent executions
SELECT
  started_at,
  status,
  records_processed,
  metadata->>'criticalCount' AS critical,
  metadata->>'warningCount' AS warning,
  metadata->>'emailSent' AS email_sent
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Price_Watchdog'
ORDER BY started_at DESC
LIMIT 10;
```

---

## Workflow Details

- **Schedule:** Daily at 6:00 AM (cron: `0 6 * * *`)
- **Email From:** noreply@techloop.com.au
- **Email To:** regan@techloop.com.au
- **Subject Format:** `🚨 Price Alert - [X] products need attention`
- **Credentials Used:**
  - Supabase Postgres (for queries and logging)
  - Elastic Email SMTP (for sending emails)

---

## Dependencies

This workflow requires:

1. **Workflow 1 (Leader Ingest)** - Must be running to keep supplier costs current
2. **Workflow 2 (WooCommerce Mirroring)** - Must be running to keep product prices current
3. **Supabase Tables:**
   - `tl_wc_products_mirror` - WooCommerce product data
   - `tl_feed_leader_raw` - Leader supplier cost data
   - `tl_workflow_executions` - Execution logging

---

## Success Checklist

- [x] Workflow created successfully
- [x] All nodes configured correctly
- [x] SQL query tested and validated
- [x] Email template formatted beautifully
- [x] Execution logging implemented
- [ ] **Workflow activated** ← YOU NEED TO DO THIS
- [ ] First email received (if alerts exist)
- [ ] Database logging verified
- [ ] Operating reliably for 1 week

---

## Full Documentation

For complete details, see:
- [[Workflow 4 - Price Watchdog - CREATED]] - Full technical documentation

---

## Support

If you encounter issues:

1. Check workflow execution logs in n8n UI
2. Verify dependencies (other workflows running)
3. Check database tables have current data
4. Review [[Workflow 4 - Price Watchdog - CREATED#Troubleshooting]] section

---

*Created: 2026-01-13 by Claude Code*
