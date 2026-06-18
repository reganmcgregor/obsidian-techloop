---
notion_page_id: null
notion_parent_id: null
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# Phase 13 — WooCommerce Decommission

Final cleanup phase of the WC→Shopify migration. The self-hosted WP/WooCommerce instance was decommissioned and all WC artefacts removed from the n8n suite and Supabase schema so that the full automation stack is natively Shopify. See [[TechLoop Shopify Migration]] for the overall phase map.

**Completed:** 2026-06-18

---

## Context and goal

Phases 9–12 rebuilt all n8n automation against Shopify while the WooCommerce instance was left running in a frozen state as a data source and rollback safety net. Phase 13 formally decommissions it:

- Remove WC mirror tables and WC-specific columns that are no longer needed
- Rename remaining columns from `wc_*` identifiers to their Shopify equivalents
- Rewrite all Supabase views to remove WC joins
- Delete the frozen WC n8n workflows (no longer rollback candidates)
- Remove any residual WC API nodes from active workflows

---

## What changed

### Supabase — dropped tables

The following tables were dropped after backing up to `_archive/phase13-wc-decom/`:

| Table | Notes |
|---|---|
| `tl_wc_products_mirror` | WC product catalogue mirror; replaced by `tl_shopify_products_mirror` |
| `tl_wc_attributes_mirror` | WC attribute definitions; mapping now carried in `tl_attribute_mapping` |
| `tl_wc_attribute_terms_mirror` | WC attribute term values |
| `tl_attribute_aliases` | WC slug alias table; superseded by `tl_attribute_mapping` |
| `tl_shopify_metafield_values_mirror` | Intermediate mirror table no longer in use |

### Supabase — renamed columns

| Table | Old column | New column | Additional changes |
|---|---|---|---|
| `tl_product_attributes` | `wc_product_id` | `shopify_product_id` | Dropped `wc_term_id` and `pushed_to_wc_at`; `wc_attribute_id` retained (FK dropped) |
| `tl_product_optimizations` | `wc_product_id` | `shopify_product_id` | — |
| `tl_onboarding_queue` | `wc_product_id` | `shopify_product_id` | Also: `wc_product_url → product_url`; `mapped_wc_* → mapped_*` |
| `tl_brand_map` | `wc_brand_*` columns | `brand_*` columns | — |

### Supabase — rewritten views

12 views were rewritten to remove WC table joins. All product-name lookups now resolve via `tl_shopify_products_mirror` joined on `supplier_product_id = stock_code`.

### n8n — deleted workflows

| Workflow | n8n ID | Reason |
|---|---|---|
| TL_Mirror_WooCommerce | `hEUnN85kXJZEG3qZ` | WC mirror — no longer needed |
| TL_Inventory_Syncer (WC) | `ZgqlMhNqh3TJMB6G` | WC sync — replaced by Shopify syncer |
| TL_Product_Detector (WC) | `rnUjGGsdD1jdaFIg` | WC detector — replaced by Shopify detector |
| Slack handler (WC) | `rDloLTYACT56kfpn` | Old WC Slack handler |
| TL_Attribute_Proposer | `hUGA1KFWBTGU8K6T` | Retired; functionality absorbed into enrichment chain |

### n8n — edited active workflows

The following live workflows had residual WC API nodes or WC-specific logic removed:

| Workflow | n8n ID |
|---|---|
| TL_Attribute_Pusher_Shopify | `s06MMF8cRDi0DXhj` |
| TL_Enrich_Attributes | `lq8960K0xVwF3Xst` |
| TL_Enrichment_Reviewer | `OarmHP6DpJvAV0Pb` |
| TL_Product_Publisher_Shopify | `pgWWBz9f6RXLXEIe` |
| TL_Product_Detector_Shopify | `fz50fq1dCmrHhazX` |

The live Slack Interaction Handler (`hikIeVV081e76pEv`) was also updated: its Create WC Attribute and Create WC Attribute Term nodes were removed.

---

## Execution summary

Work was carried out in a single session on 2026-06-18, following a pre-agreed step plan:

1. Safety snapshot — Supabase table backups exported to `_archive/phase13-wc-decom/`
2. Supabase migration — column renames, FK drops, index updates, 12 view rewrites
3. Active workflow edits — WC nodes removed from 5 workflows + Handler
4. Frozen WC workflow deletion — 4 WC originals + TL_Attribute_Proposer deleted
5. Reactivation — all affected active workflows deactivated and reactivated to re-register triggers
6. End-to-end verification (see below)

---

## Verification

End-to-end test after decommission:

- **TL_Enrich_Attributes** exec `57884` — ran successfully, wrote new attribute proposals to `tl_product_attributes`
- **TL_Attribute_Pusher_Shopify** exec `57890` — pushed metafield for product `8903460749510` successfully; metafield confirmed written in Shopify Admin

No WC-related errors observed in either execution.

### Final verification — 2026-06-18

A full read-only sweep of all 13 active n8n workflows and the live Supabase schema was completed on 2026-06-18. Result: **zero functional WC references** found.

- No `wp-json/wc/` HTTP calls in any workflow
- No `wooCommerceApi` credential in use
- No `tl_wc_*` table or view queries in any active workflow
- All 5 target workflows confirmed deleted; all other frozen WC workflows confirmed inactive
- Zero `tl_wc_*` tables or views in Supabase
- No `pushed_to_wc` rows remaining in any table

Phase 13 is fully closed.

### Cosmetic WC-named leftovers (deferred to enrichment project)

The items below carry WC-derived names but have **no runtime WC dependency** — they are mapping keys, a Slack action ID, JS variable names, or dormant unused columns. They do not affect the live stack and are deferred to the [[Phase 14 — Enrichment Service Rebuild]] project.

- `tl_attribute_mapping.wc_attribute_id` and `wc_slug` — mapping-key columns (no FK, no WC API call)
- `tl_product_attributes.wc_attribute_id` — retained mapping reference
- `tl_proposed_attributes.wc_attribute_id` — retained mapping reference
- Slack `action_id: wc_publish_product` — routes to Shopify; rename is cosmetic only
- JS variable names in active workflow code (no external WC calls)
- Dormant unused columns: `tl_category_map.wc_category_id`/`wc_category_name`, `tl_product_supplier_status.wc_id`, `tl_sync_log.wc_id`

---

## Known follow-ups

These items were identified during Phase 13 and deferred for later resolution. They do not block the current live stack.

1. **`tl_attribute_mapping` display-name column missing** — Views currently reference `wc_slug` for display names. A `display_name` column should be added to `tl_attribute_mapping` and views updated to use it.

2. **`vw_products_needs_sync` + `vw_discontinued_products` lost stock columns** — These views lost their inventory columns when WC tables were dropped (no equivalent column on the products mirror). Both views are currently unused; if reactivated they should be repointed to `tl_shopify_inventory_levels_mirror`.

3. **Pre-existing metafield type drift** — One row in `tl_attribute_mapping` has type `list` vs the expected scalar type, causing an `INVALID_TYPE` error on one attribute push. Requires a targeted fix to the mapping row.

4. **`TL_Product_Publisher_Shopify` schedule vs docs mismatch** — The workflow is `active: true` on a 5-minute schedule, which contradicts documentation stating it should be inactive pending user go-live decision. Confirm intended state and update docs or workflow accordingly.

---

## Backups

Pre-decommission Supabase table dumps are stored at:

```
_archive/phase13-wc-decom/
```

This directory is Obsidian-only (not synced to Notion). Retain until the follow-ups above are resolved and the stack has been stable for at least 30 days.
