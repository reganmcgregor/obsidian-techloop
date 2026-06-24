# WC-naming cleanup — pending runbook (blocked on labgregor outage)

**Status 2026-06-24:** `labgregor` unreachable (SSH port 22 timeout + n8n MCP `NO_RESPONSE`).
Everything below is **authorised by the user but NOT yet applied** — execute in order when
the host returns. The core column migration (`wc-naming-cleanup-2026-06-24.sql`) already
ran and is verified; this runbook covers the remaining items.

## 0. Connectivity gate
- `ssh reganmcgregor@labgregor 'echo ok'` succeeds, and
- `mcp__n8n-mcp__n8n_health_check` returns OK (needed for the workflow edits in step 3).

## 1. Part A — comment fixes  *(SQL ready in `wc-naming-cleanup-2026-06-24-followup.sql`, Part A)*
Zero dependency. Rewrites the 4 live COMMENTs still saying "WooCommerce"
(`tl_category_map`, `tl_brand_map`, `tl_onboarding_queue.mapped_product_type`,
`vw_unmapped_categories` → will become `tl_vw_unmapped_categories` in step 3).

## 2. Part B — rename stale index + constraint on `tl_product_optimizations`
*(SQL ready in followup.sql, Part B — currently commented out)*
- **PRECONDITION (the check I promised):** via n8n MCP, confirm NO workflow upserts
  `tl_product_optimizations` using `ON CONFLICT ON CONSTRAINT
  tl_product_optimizations_wc_product_id_key`. Column-based `ON CONFLICT
  (shopify_product_id)` is unaffected. Inspect the writers (Enrich `lq8960K0xVwF3Xst`,
  Reviewer `OarmHP6DpJvAV0Pb`, any Optimizer).
- If clear: `ALTER INDEX idx_tl_product_optimizations_wc_id RENAME TO
  idx_tl_product_optimizations_shopify_product_id;` and `ALTER TABLE
  tl_product_optimizations RENAME CONSTRAINT tl_product_optimizations_wc_product_id_key
  TO tl_product_optimizations_shopify_product_id_key;`

## 3. View namespacing — `vw_*` → `tl_vw_*` (ALL views)
**Decisions (user, 2026-06-24):** convention `tl_vw_<name>` (prepend `tl_`, keep `vw_`
marker); scope = every view.

**Mechanics:** `ALTER VIEW … RENAME` is metadata-only and auto-cascades to *dependent
views* (Postgres binds them by OID). The ONLY thing that breaks is **external references
by name** — i.e. n8n workflow SQL strings (`FROM vw_x` / `JOIN vw_x`). Those are plain
text and must be edited in lockstep.

**Steps:**
1. **Enumerate** every view + matview:
   `SELECT c.relname, c.relkind FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
    WHERE n.nspname='public' AND c.relkind IN ('v','m') ORDER BY 1;`
   Build the rename map `vw_X → tl_vw_X`. Known so far (confirm/extend from the query):
   `vw_new_products_for_onboarding`, `vw_new_products_for_shopify_onboarding`,
   `vw_unmapped_categories`, `vw_attribute_coverage`, `vw_proposed_attributes`,
   `vw_pending_attribute_mappings`, `vw_onboarding_queue_summary`,
   `vw_onboarding_queue_recent`, `vw_products_needs_sync`, `vw_discontinued_products`.
2. **Map workflow references** (n8n MCP): for each view name, find every workflow whose
   Postgres-node SQL contains it. Known: `TL_Product_Detector_Shopify` (`fz50fq1dCmrHhazX`)
   → `SELECT * FROM vw_new_products_for_shopify_onboarding`. Sweep all ~13 active workflows
   for `vw_` to catch the rest.
3. **Rename** each view in one transaction: `ALTER VIEW vw_X RENAME TO tl_vw_X;`
   (matviews: `ALTER MATERIALIZED VIEW …`). Dependent views auto-follow.
4. **Update workflows** in lockstep via `n8n_update_partial_workflow` — replace each
   `vw_X` with `tl_vw_X` in the affected node SQL. Validate (`n8n_validate_workflow`) and
   re-fetch (`n8n_get_workflow`) to confirm.
5. **Verify:** no `vw_*` views remain
   (`SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
     WHERE n.nspname='public' AND c.relname LIKE 'vw\_%' AND c.relkind IN ('v','m');` → 0);
   each renamed view still `pg_get_viewdef`s cleanly; `SELECT … LIMIT 1` from a couple;
   no workflow still contains `vw_`.

## 4. Docs + memory after applying
- Update `Database Schema - Product Automation.sql/.md` view names → `tl_vw_*`, and the
  relationship diagrams / sample queries (`SELECT * FROM vw_…`).
- Update `phase13_status.md` cleanup note: comments fixed, index/constraint renamed, all
  views namespaced `tl_vw_*`.
- Consider a Notion Decisions Log entry (per CLAUDE.md task-tracking).

## Rollback
All steps are metadata-only and reversible: re-run the `ALTER … RENAME` statements with the
names swapped; revert the workflow SQL edits via workflow version history
(`n8n_workflow_versions`).
