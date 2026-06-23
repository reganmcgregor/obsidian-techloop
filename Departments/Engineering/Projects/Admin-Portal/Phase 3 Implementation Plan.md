# Phase 3 Implementation Plan: Workflow Parity & System Mirrors

This phase focuses on reaching feature parity with the existing n8n/Slack workflows and integrating the full TechLoop data ecosystem into the Admin Portal. 

> [!NOTE]
> *Note on Enrichment: The enrichment workflow is currently being rebuilt, so all enrichment-related UI has been excluded from this plan.*

## User Review Required

> [!IMPORTANT]
> Please review the open questions below to ensure the implementation matches your workflow preferences. Once approved, I will begin execution and also mirror this finalized plan to your Obsidian Vault.

## Open Questions

> [!WARNING]
> 1. **Category Mappings Table:** The current table schema (`tl_category_map`) uses `wc_category_id` and `wc_category_name` (WooCommerce), but your workflows sync to Shopify. Are you planning to rename these columns to `shopify_collection_id`, or should we stick to the `wc_` naming convention in the UI for now?
> 2. **Shopify Mirror Refresh:** Should the Shopify Mirrors view include a manual "Force Refresh" button to trigger the n8n sync workflow, or will it remain strictly read-only?

## Proposed Changes

### 1. Queue Upgrades & Link Parity

Bring the Onboarding Queue up to parity with the Slack workflow by exposing direct product links and fixing the categories data source.

#### [MODIFY] [src/app/queue/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/queue/page.tsx)
- Add `shopify_product_url` and `product_url` to the Supabase select query.

#### [MODIFY] [src/app/queue/data-table.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/queue/data-table.tsx)
- Add a dedicated "Links" column displaying the Shopify logo (if `shopify_product_url` exists) and the Supplier link (if `product_url` exists).

#### [MODIFY] [src/app/categories/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/categories/page.tsx) & [actions.ts](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/categories/actions.ts)
- Switch the data source from the placeholder `tl_category_rules` to the actual `tl_category_map` table so real data populates the view.
- Update columns to match the actual schema (`supplier_category_name`, `wc_category_id`, `is_blocked`, `auto_approve`).

---

### 2. Brands Management

Create a dedicated interface for managing the `tl_brand_map`.

#### [NEW] [src/app/brands/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/brands/page.tsx) & [data-table.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/brands/data-table.tsx)
- Build a data table displaying `supplier_brand_name`.
- Add mapping fields for `brand_name` and `brand_slug`.
- Include toggles for `is_blocked` and display `block_reason`.

---

### 3. Suppliers Dashboard

Create a control center for supplier configurations and sync statuses (`tl_suppliers`).

#### [NEW] [src/app/suppliers/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/suppliers/page.tsx)
- Display all suppliers (`name`, `code`, `is_active`).
- Show the `last_sync_at` timestamp and `sync_interval_minutes`.
- Add a toggle to quickly activate/deactivate a supplier feed.

---

### 4. Shopify Mirrors View

A read-only dashboard to inspect the current state of Shopify directly within the portal.

#### [NEW] [src/app/mirrors/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/mirrors/page.tsx)
- Query `tl_shopify_products_mirror`.
- Display a paginated table of live products with columns for `title`, `vendor`, `status`, and `updated_at_remote`.
- Allow expanding a row to view the `raw_json` payload stored in the mirror.

## Verification Plan

### Automated Tests
- Run `npm run build` to verify types.

### Manual Verification
- Verify the Categories page now correctly loads data from `tl_category_map`.
- Confirm external Shopify links open correctly from the Queue.
- Check that Suppliers and Brands correctly update their respective tables.
