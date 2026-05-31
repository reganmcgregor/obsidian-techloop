---
notion_page_id: 3558c3d3-c03e-8175-802e-d409109bbe66
notion_parent_id: 3558c3d3-c03e-8120-ac7a-eda8804ac5ae
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: playbook
synced_at: 2026-05-03T16:38:58Z
---

# Schema Deployment

> Process for applying database schema changes to Supabase / WooCommerce / mirror tables.

## When to use

Adding/altering columns on `tl_*` tables, creating or modifying views or indexes, adding functions or triggers. Also use when a new supplier integration requires new raw feed tables.

## Steps

1. **Draft the SQL** in `Departments/Engineering/Context/Infrastructure/` (alongside the existing `.sql` file). Keep DDL in source control under that directory.
2. For non-trivial changes (column drops, type changes): test on a snapshot first — export via Supabase Studio or `pg_dump`.
3. **Check for active runs** — query `tl_workflow_executions WHERE status = 'running'` and wait for completion before altering tables used by live workflows.
4. **Apply the migration** via Supabase MCP (`apply_migration`) or psql:
   ```bash
   ssh 192.168.1.111
   docker exec supabase-db psql -U postgres -d postgres -f /path/to/migration.sql
   ```
5. **Verify** — run a sample query against the affected table/view; confirm row counts and column types are correct.
6. **Update affected n8n workflows** — if column names changed or new fields need to be written, update workflows immediately using [[n8n Workflow Update Process]].
7. **Check `tl_sync_log`** on the next scheduled run to confirm the workflow writes correctly to the new schema.
8. **Update the schema doc** at `Departments/Engineering/Context/Infrastructure/Database Schema - TechLoop Inventory.md` to reflect the change.

## Notes / gotchas

- **RLS is OFF on all `tl_*` tables** — no row-level security to configure.
- **`tl_wc_products_mirror` PK is `wc_id` not `id`** — all joins must use `pm.wc_id = pa.wc_product_id`.
- `/opt` requires sudo on 192.168.1.111 — use `~/` (home dir) for any user-writable files.
- **Supabase REST upserts**: use `Prefer: resolution=merge-duplicates` header.
- **Supabase REST from n8n**: use `http://supabase-kong:8000/rest/v1/<table>` (both on the `proxy` Docker network). `host.docker.internal:8001` times out due to iptables.
- **Schema philosophy**: Raw over Modelled — store raw feed data and WC data; compute diffs in views (`vw_products_needs_sync`, `vw_discontinued_products`).
- All timestamp columns must use `TIMESTAMP WITH TIME ZONE` (UTC storage).

## Last reviewed

2026-05-03 by Regan
