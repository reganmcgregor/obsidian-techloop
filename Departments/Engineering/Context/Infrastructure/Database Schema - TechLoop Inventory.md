# Database Schema - TechLoop Inventory Automation

**Status:** ✅ Production Ready - Deployed 2026-01-13

See the complete SQL schema file: `Database Schema - TechLoop Inventory.sql`

---

## Quick Reference

### Tables (5)
1. **tl_suppliers** - Master supplier list (1 row: Leader)
2. **tl_feed_leader_raw** - Raw Leader XML feed (50+ fields, 5+ products)
3. **tl_wc_products_mirror** - WooCommerce products mirror (6+ products)
4. **tl_sync_log** - Audit trail of syncs
5. **tl_workflow_executions** - Workflow execution tracking

### Views (3)
1. **vw_products_needs_sync** - Products with stock/cost differences (6 products)
2. **vw_low_margin_products** - Products with margins < 10% (0 products)
3. **vw_discontinued_products** - Products not in feed 2+ hours (6 products)

### Functions (2)
1. **sanitize_availability(text)** - Convert CALL/POA to 0
2. **update_updated_at_column()** - Auto-update timestamps

### Indexes (29 total)
- Priority 1: 8 indexes (essential for JOINs)
- Priority 2: 10 indexes (performance)
- Priority 3: 3 indexes (advanced queries)
- GIN: 4 indexes (JSONB searches)
- Unique: 6 indexes (PKs and constraints)

---

## Key Design Principles

1. **Raw over Modelled** - Store ALL data, preserve source structure
2. **Snake_Case** - Column names match source fields (e.g., `stock_code`)
3. **TIMESTAMP WITH TIME ZONE** - All timestamps in UTC
4. **JSONB for Extensibility** - Future-proof with raw_meta_data, raw_xml
5. **Audit Everything** - 100% traceability in tl_sync_log

---

## Critical Design Decisions

### Availability: Raw + Sanitized
- `availability_total_raw` (VARCHAR) - Original: "42", "CALL", "POA"
- `availability_total` (INTEGER) - Sanitized: 42, 0, 0
- **Why?** Preserve source data while enabling reliable queries

### Incremental WooCommerce Sync
- Use `modified_after` parameter
- 95% reduction in API calls (2 calls vs 50)
- Only fetch products changed since last sync

### Decoupled Workflows
- Workflow 1: Leader Ingest (independent)
- Workflow 2: WC Mirroring (independent)
- Workflow 3: Sync Differences (depends on both)
- **Why?** Leader data stays fresh even if WC API fails

---

## Verification Status

✅ All 5 tables deployed
✅ All 3 views queryable
✅ All 29 indexes created
✅ All 2 functions working
✅ All 3 triggers active
✅ sanitize_availability() tests pass
✅ All timestamps use TIMESTAMP WITH TIME ZONE
✅ Raw + sanitized availability columns exist
✅ 8 JSONB columns deployed

---

## Sample Data

- **Suppliers:** 1 (Leader)
- **Leader Products:** 5
- **WC Products:** 6
- **Products Needing Sync:** 6
- **Low Margin Products:** 0
- **Discontinued Products:** 6 (test data)

---

## Related Files

- **SQL Schema:** `Database Schema - TechLoop Inventory.sql`
- **JavaScript Utilities:** `n8n-utilities.js`
- **Project Plan:** `~/.claude/plans/robust-forging-blossom.md`

---

## Tags

#infrastructure #database #automation #inventory #leader #supabase

