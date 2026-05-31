> [!info] Part of [[Master Note - Inventory Automation]]

# Quick Reference - Workflow URLs

Quick access links to all TechLoop Inventory Automation workflows in n8n.

---

## Workflow 1: Leader Ingest

**Name:** TL_Ingest_Leader_Feed
**ID:** `mUkhS0BfN7Lq6EsS`
**Status:** 🔴 Inactive (Ready for activation)
**Schedule:** Every 2 hours at :00

**Direct Link:** https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS

**Purpose:** Fetches Leader XML feed every 2 hours and upserts to `tl_feed_leader_raw`

**Documentation:** [[Workflow 1 - Leader Ingest - DEPLOYED]]

---

## Workflow 2: WooCommerce Mirroring

**Status:** ⏳ Not yet built

**Planned Schedule:** Every 4 hours at :15

**Purpose:** Mirror WooCommerce products to `tl_wc_products_mirror` (incremental with `modified_after`)

---

## Workflow 3: Inventory Syncer (Diff Engine)

**Status:** ⏳ Not yet built

**Planned Schedule:** Every 3 hours at :30

**Purpose:** Detect differences and sync inventory/cost from Leader to WooCommerce

---

## Workflow 4: Price Watchdog

**Status:** ⏳ Not yet built

**Planned Schedule:** Daily at 6:00 AM

**Purpose:** Detect low margin products and send email alerts

---

## n8n Access

**Web UI:** https://n8n.reganmcgregor.com.au
**Server:** Ubuntu 192.168.1.111
**Service:** `systemctl status n8n`

---

## Supabase Access

**URL:** https://supabase.reganmcgregor.com.au
**MCP:** Available via `mcp__supabase__*` tools

**Key Tables:**
- `tl_feed_leader_raw` - Leader product feed (raw ingestion)
- `tl_wc_products_mirror` - WooCommerce products mirror
- `tl_workflow_executions` - Workflow execution logs
- `tl_sync_log` - Inventory sync audit trail
- `tl_suppliers` - Master supplier list

---

## Email Reports

**Workflow 1 Reports:** regan@techloop.com.au
**From:** noreply@techloop.com.au
**Frequency:** After every execution (every 2 hours)

---

## Monitoring Queries

### Check Latest Executions
```sql
SELECT workflow_name, started_at, status, duration_seconds, total_items
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Ingest_Leader_Feed'
ORDER BY started_at DESC
LIMIT 5;
```

### Check Daily Stats
```sql
SELECT
  DATE(started_at) as date,
  COUNT(*) as runs,
  ROUND(AVG(duration_seconds), 0) as avg_duration,
  SUM(total_items) as products
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Ingest_Leader_Feed'
  AND started_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(started_at)
ORDER BY date DESC;
```

---

## Quick Actions

### Activate Workflow 1
1. Open: https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS
2. Click **Activate** toggle (top right)
3. Wait for email confirmation on next :00 hour

### Test Workflow 1 Manually
1. Open: https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS
2. Click **Test workflow** button
3. Monitor execution in real-time
4. Check email for report

### View Execution History
1. Open: https://n8n.reganmcgregor.com.au/executions
2. Filter by workflow: `TL_Ingest_Leader_Feed`
3. Click execution to view details

---

## Related Documentation

- [[Master Note - Inventory Automation]] - Project overview
- [[Workflow 1 - Leader Ingest - DEPLOYED]] - Full documentation
- `/Users/regan.mcgregor/.claude/plans/robust-forging-blossom.md` - Implementation plan

---

**Last Updated:** 2026-01-13
