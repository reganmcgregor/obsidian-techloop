---
notion_page_id: 3558c3d3-c03e-8186-ab19-d035e22af89b
notion_parent_id: 3558c3d3-c03e-8120-ac7a-eda8804ac5ae
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: playbook
synced_at: 2026-05-03T16:39:31Z
---

# Incident Response

> Triage and recovery steps for production incidents (broken sync, scraper failure, store outage).

## When to use

A Slack alert fires from a workflow, inventory has not updated for >2 hours, a scraper job is failing, WooCommerce is out of sync, or Slack notifications have stopped.

## Severity

| Level | Definition | SLA |
|-------|-----------|-----|
| P0 | Store down / checkout broken | Notify customer support immediately; fix now |
| P1 | Inventory sync broken (>2 hrs) | 4-hour fix SLA |
| P2 | Degraded (scraper slow, minor data gap) | Next business day |

## Steps

### 1. Slack alert fires from a workflow
1. Check `tl_workflow_executions` for the most recent failed run (`status = 'error'`).
2. Open the n8n execution log for that run to identify the failing node.
3. Check `tl_sync_log` for related error rows.
4. Fix the root cause, then trigger a manual one-shot run via the workflow UI to confirm recovery.

### 2. Inventory not syncing (no products updated >2 hours)
1. **First check:** is the Schedule Trigger still active? The most common cause is silent deregistration after a partial workflow update. If inactive, reactivate via REST API (see [[Deploy Checklist]]).
2. Check Leader feed availability — attempt to fetch the feed URL directly.
3. Check Supabase connection — run `docker exec supabase-db psql -U postgres -d postgres -c "SELECT 1;"` on the server.
4. Check `tl_workflow_executions` — if there are rows with `status = 'running'` for >30 minutes, the run is hung; stop and restart.

### 3. Scraper failure
1. Check Crawl4AI container: `curl http://192.168.1.111:11235/health` — container may be down.
2. Restart if needed: `ssh 192.168.1.111` → `docker restart crawl4ai`.
3. Query `tl_manufacturer_raw_data` for `scrape_status` values to identify which products are affected.
4. Re-trigger scraping workflow manually for failed products.

### 4. WooCommerce out of sync
1. Confirm WC API credentials are valid — credential ID `vMZW4D7gnzSYVw0g`; test via a simple GET to `www.techloop.com.au`.
2. Check `vw_products_needs_sync` view — how many products are queued?
3. Verify joins use `wc_id` (not `id`) in `tl_wc_products_mirror`.
4. Run the Inventory Syncer workflow manually to force a sync pass.

### 5. Slack notifications not firing
1. Confirm Bot Token credential `DvtYgxYgI1p5Q3QX` ("TechLoop Automation") is valid in n8n.
2. Check the Slack node has a `text` fallback parameter — Slack returns `no_text` without it.
3. Confirm block JSON is wrapped in an object: `{ "blocks": [...] }` not a bare array.
4. Check n8n execution logs for the Slack node output — look for API error detail.

## Notes / gotchas

- **Prefer a manual one-shot run** over a re-deploy when recovering. Re-deploying risks silent parameter loss.
- After any fix that touches a scheduled workflow, always deactivate+reactivate the Schedule Trigger.
- `tl_workflow_executions` and `tl_sync_log` are the primary audit trail — check these before escalating.

## Last reviewed

2026-05-03 by Regan
