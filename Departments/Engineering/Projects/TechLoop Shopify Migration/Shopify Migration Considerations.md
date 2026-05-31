# Shopify Migration Considerations

> **Historical reference (Apr–early May 2026).** Pre-decision impact assessment. Current decisions, statuses, and live store state live in [[TechLoop Shopify Migration]]. Cost figures, plan recommendations, and "options" lists in this document have been superseded — refer to the main spec for what was actually chosen and executed.

This document assesses the impact of migrating TechLoop from WooCommerce to Shopify, with a focus on the existing n8n automation ecosystem, Supabase database layer, and HITL workflows.

---

## 1. Current WooCommerce Touchpoints

Every component below directly integrates with the WooCommerce REST API:

| Component | Integration Type | What It Does |
|-----------|-----------------|--------------|
| **TL_Mirror_WooCommerce** | WC REST API (GET) | Incremental product sync via `modified_after`; extracts ACF fields |
| **TL_Inventory_Syncer** | WC REST API (PUT) | Pushes stock/cost/discontinuation updates to WC products |
| **TL_Price_Watchdog** | Supabase views (reads WC mirror) | Scans for low-margin products using mirrored WC data |
| **TL_Product_Detector** | Supabase views (reads WC mirror) | Detects new Leader products not yet in WC |
| **TL_Product_Publisher** | WC REST API (POST) | Creates draft products, uploads media, sets ACF fields |
| **TL_Mirror_WC_Attributes** | WC REST API (GET) | Mirrors attribute definitions and terms to Supabase |
| **TL_Attribute_WC_Pusher** | WC REST API (PUT) | Pushes approved attribute mappings to WC products |
| **TL_Import_WC_Brands** | WC REST API (GET) | Imports brand taxonomy |
| **TL_Seed_Qdrant_Titles** | WC REST API (GET) | Reads existing product titles for RAG seeding |
| **TL_Seed_Qdrant_Short_Descriptions** | WC REST API (GET) | Reads existing short descriptions for RAG seeding |
| **`tl_wc_products_mirror`** | Supabase table | Stores mirrored WC product state including ACF fields |
| **`tl_wc_attributes_mirror`** | Supabase table | Mirrors WC global attribute definitions |
| **`tl_wc_attribute_terms_mirror`** | Supabase table | Mirrors WC attribute terms |
| **`tl_sync_log`** | Supabase table | Logs every WC change (references WC product IDs) |

---

## 2. API Differences — WooCommerce vs Shopify

| Concept | WooCommerce REST API | Shopify Admin API |
|---------|---------------------|-------------------|
| **Authentication** | Basic Auth (consumer key/secret) | OAuth 2.0 or private app access token |
| **Products endpoint** | `/wp-json/wc/v3/products` | `/admin/api/2024-01/products.json` (versioned) |
| **Product status** | `draft`, `publish`, `pending` | `draft`, `active`, `archived` |
| **Custom fields** | ACF meta fields via `meta_data` array | Metafields with namespace/key/type — requires metafield definitions |
| **Attributes** | Global attributes with taxonomies + per-product attributes | Options (colour/size for variants) OR metafields (specs) — no direct equivalent of WC global attributes |
| **Categories** | Hierarchical taxonomy via API | Collections (smart or manual) — flat, not hierarchical |
| **Brands** | Custom taxonomy (via plugin) | Metafield or Shopify Markets — no native brand taxonomy |
| **Media upload** | `POST /media` then attach ID to product | `POST /products/{id}/images` with `src` URL or base64 — simpler but different flow |
| **Incremental sync** | `modified_after` parameter | `updated_at_min` parameter — similar concept |
| **Bulk operations** | `/batch` endpoint (100 items) | GraphQL Bulk Operations or REST batch — more powerful but different pattern |
| **Webhooks** | WC webhooks (limited reliability) | Shopify webhooks (more reliable, mandatory HMAC verification) |
| **Rate limits** | None (self-hosted) | 2 requests/second (REST) or 1000 points/second (GraphQL) |

### Key API Concerns

- **Rate limiting**: WooCommerce is self-hosted with no rate limits. Shopify enforces strict limits — workflows that batch-update products (Syncer, Pusher) will need throttling logic.
- **GraphQL vs REST**: Shopify's GraphQL API is significantly more efficient for bulk reads. Worth considering for Mirror and Syncer workflows to reduce API calls.
- **API versioning**: Shopify versions its API quarterly and deprecates old versions. Requires ongoing maintenance to stay current.

---

## 3. n8n Workflow Changes Required

### High Impact (Major Rewrite)

| Workflow | Current | Required Changes |
|----------|---------|-----------------|
| **TL_Mirror_WooCommerce** | WC node with `modified_after` + ACF extraction | Replace WC node with Shopify node or HTTP Request. Rewrite ACF field extraction to read metafields instead. Update `tl_wc_products_mirror` schema to match Shopify product structure. |
| **TL_Inventory_Syncer** | Pushes stock/cost updates via WC API | Replace WC update calls with Shopify inventory API (`/inventory_levels/set.json` for stock). Cost/COGS handling differs — Shopify uses `cost` field on variants. Discontinuation logic needs rethinking (Shopify uses `archived` status). |
| **TL_Product_Publisher** | Creates WC draft with ACF fields, uploads media, sets attributes | Complete rewrite of product creation payload. ACF fields → metafields. Attributes → metafields or variant options. Image upload flow changes (Shopify can sideload from URL directly). Category assignment → collection membership (separate API call). |
| **TL_Attribute_WC_Pusher** | Pushes attribute terms to WC global attributes | Shopify has no equivalent of WC global attributes for specs. Needs redesign — likely push to product metafields instead. Attribute mirror concept needs rethinking. |
| **TL_Mirror_WC_Attributes** | Reads WC attribute definitions/terms | Replace with metafield definitions API or remove entirely if attributes become metafields. |

### Medium Impact (Moderate Changes)

| Workflow | Current | Required Changes |
|----------|---------|-----------------|
| **TL_Product_Detector** | Reads `tl_wc_products_mirror` via Supabase views | No direct WC API calls, but depends on mirror table schema. Views need updating once mirror schema changes. Detection logic (matching by `acf_supplier_product_id`) needs equivalent Shopify metafield lookup. |
| **TL_Price_Watchdog** | Reads Supabase views derived from WC mirror | Same as above — views need updating. Pricing model may differ if Shopify's compare-at-price replaces RRP handling. |
| **TL_Slack_Interaction_Handler** | Updates WC products on button clicks (publish, etc.) | Replace WC API calls in publish/update branches with Shopify equivalents. Button payloads may need different IDs. |
| **TL_Import_WC_Brands** | Reads WC brand taxonomy | Shopify has no brand taxonomy. Either use a metafield-based approach or a Shopify app. Workflow may become unnecessary if brands are handled via metafields during product creation. |

### Low Impact (Minor or No Changes)

| Workflow | Impact |
|----------|--------|
| **TL_Ingest_Leader_Feed** | No change — reads supplier XML, writes to Supabase only |
| **TL_URL_Discoverer** | No change — reads Supabase, writes to Supabase |
| **TL_Scraper** | No change — Crawl4AI scrapes manufacturer sites, not TechLoop |
| **TL_Enrich_Attributes** | No change — LLM extraction from manufacturer data |
| **TL_Attribute_Proposer** | No change — Slack HITL for attribute review |
| **TL_Enrichment_Reviewer** | No change — Slack notifications for pending attributes |
| **TL_Queue_Reviewer** | Minor — reads Supabase, sends to Slack. May need updated field names. |
| **TL_Seed_Qdrant_*** | One-time re-run needed after migration to seed from Shopify data |

---

## 4. Database / Supabase Impact

### Tables Requiring Schema Changes

| Table | Changes Needed |
|-------|---------------|
| **`tl_wc_products_mirror`** | Rename or create new `tl_shopify_products_mirror`. Replace WC-specific columns: `wc_id` → `shopify_id` (bigint), `acf_supplier_product_id` → metafield equivalent, remove `acf_*` columns, add Shopify-specific fields (`variant_id`, `inventory_item_id`). |
| **`tl_wc_attributes_mirror`** | Likely replaced with `tl_shopify_metafield_definitions` or removed entirely. |
| **`tl_wc_attribute_terms_mirror`** | Same as above — metafield values don't have the same taxonomy structure. |
| **`tl_sync_log`** | Update `wc_product_id` references to `shopify_product_id`. Historical WC logs remain valid but new entries use Shopify IDs. |
| **`tl_onboarding_queue`** | Currently references WC product ID after publishing. Update to Shopify product ID. |
| **`tl_product_optimizations`** | Same — update WC references to Shopify. |
| **`tl_product_attributes`** | Review status flow unchanged, but the "push to store" step targets metafields instead of WC attributes. |

### Views Requiring Updates

All views that JOIN on `tl_wc_products_mirror` need updating:
- `vw_products_needs_sync`
- `vw_low_margin_products`
- `vw_discontinued_products`
- `vw_new_products_for_onboarding`
- `vw_onboarding_queue_summary`
- `vw_unmapped_categories`
- `vw_products_need_description`
- `vw_products_need_scraping`
- `vw_products_need_attribute_extraction`

### What Stays the Same

- `tl_suppliers` — supplier data, no WC dependency
- `tl_feed_leader_raw` — raw XML feed storage
- `tl_category_map` — needs updating for Shopify collection IDs but structure is fine
- `tl_brand_map` — same concept, different target
- `tl_manufacturer_raw_data` — scraping data, no WC dependency
- `tl_attribute_aliases` — alias resolution is platform-agnostic
- `tl_description_models` — LLM config, no WC dependency
- `tl_workflow_executions` — execution tracking, no WC dependency

---

## 5. Feature Gaps and Differences

### ACF Custom Fields → Shopify Metafields

| WC (ACF)                   | Shopify Equivalent                              | Notes                                          |
| -------------------------- | ----------------------------------------------- | ---------------------------------------------- |
| `acf_supplier_name`        | Product metafield `custom.supplier_name`        | Must create metafield definition first         |
| `acf_supplier_product_id`  | Product metafield `custom.supplier_product_id`  | Critical — used for matching in detection/sync |
| `acf_supplier_dbp_ex_gst`  | Product metafield `custom.cost_ex_gst`          | Shopify also has native `cost` on variants     |
| `acf_supplier_dbp_inc_gst` | Product metafield `cuj innn ustom.cost_inc_gst` | —                                              |
| `acf_supplier_mpn`         | Product metafield `custom.mpn`                  | —                                              |
| `acf_supplier_barcode`     | M mm kk mm ok to nnnuuyhhhhuuhjnn               | Shopify has native barcode support             |
| `_wc_gtin`                 | Variant `barcode` field (native)                | Same as above                                  |

### Global Attributes → ???

This is the **biggest structural difference**. WooCommerce has a proper attribute taxonomy system (global attributes with terms, per-product assignment). Shopify does not.

**Options for product specs in Shopify:**
1. **Product metafields** — key/value pairs, filterable with Online Store 2.0 themes. Closest equivalent but no shared taxonomy across products.
2. **Product tags** — flat, no structure. Poor fit for structured specs.
3. **Variant options** — limited to 3 options, intended for purchasable variants (size/colour). Not suitable for specs.
4. **Shopify app** (e.g., Custom Fields, Accentuate) — third-party solutions that add structured specs.

**Recommendation**: Product metafields with metafield definitions. The `tl_attribute_aliases` system and HITL approval flow can still work — just push to metafields instead of WC attributes.

### Other Differences

| Feature | WooCommerce | Shopify |
|---------|-------------|---------|
| **Categories** | Hierarchical taxonomy (parent/child) | Collections — flat, smart (rule-based) or manual |
| **Product drafts** | `status: draft` → `publish` | `status: draft` → `active`. Shopify also has "hidden from sales channels" |
| **Image processing** | Upload to WP Media Library, get ID, attach | Provide URL or base64, Shopify hosts it. Simpler but less control. |
| **Pricing / Tax** | Price inc/exc GST, tax classes | Tax handled at checkout via Shopify Tax. Price is always tax-exclusive in API. |
| **Compare-at price** | `regular_price` + `sale_price` | `price` + `compare_at_price` — similar concept |
| **Variants** | Variations (child products) | Variants on same product (max 3 options, 100 variants) |
| **SEO** | Yoast/RankMath plugins | Built-in SEO fields (`meta_title`, `meta_description`, `handle` for URL) |

---

## 6. Slack HITL Impact

The Slack interaction handler (`TL_Slack_Interaction_Handler`) routes button callbacks to different branches. Most branches update Supabase tables (low impact), but some call WC directly:

| Action | Current Target | Shopify Change |
|--------|---------------|----------------|
| `approve_product` | Updates `tl_onboarding_queue` in Supabase | No change |
| `reject_product` | Updates `tl_onboarding_queue` in Supabase | No change |
| `wc_publish_product` | Calls WC API to change status to `publish` | Replace with Shopify API call to set `status: active` |
| `approve_new_attribute` | Creates attribute in WC via API | Replace with metafield definition creation in Shopify |
| `map_to_existing` | Maps to WC attribute (Supabase only) | No change (Supabase operation) |
| `approve_new_term` | Creates term in WC attribute | Replace with — metafield values don't need pre-creation |
| `approve_attr_mapping` | Updates Supabase | No change |
| Google Search | External link | No change |

**Net impact**: 3 of 8 button actions need Shopify API replacements.

---

## 7. Enrichment Pipeline Impact

| Component | Impact | Details |
|-----------|--------|---------|
| **URL Discovery** | None | Reads Leader feed data, writes to Supabase |
| **Scraping (Crawl4AI)** | None | Scrapes manufacturer sites, not the store |
| **Attribute Extraction (LLM)** | None | Processes manufacturer markdown, platform-agnostic |
| **Attribute Proposer (HITL)** | None | Slack-based review, Supabase storage |
| **Attribute WC Pusher** | **High** | Must push to Shopify metafields instead of WC attributes. Different API, different data model. |
| **Description Enrichment** (planned) | **Low** | Output is HTML — works for both WC and Shopify product descriptions |
| **Image Enrichment** (planned) | **Medium** | Upload mechanism differs. Shopify can sideload from URL which is actually simpler than WC's media library flow. |

---

## 8. Infrastructure Changes

### n8n Nodes

| Current | Replacement |
|---------|------------|
| WooCommerce node (native n8n) | Shopify node (native n8n) or HTTP Request node for Shopify Admin API |

n8n has a native Shopify node, but it may not cover all endpoints (metafields, inventory levels, bulk operations). Expect to use HTTP Request nodes for advanced operations, similar to how some WC operations already use HTTP Request.

### Credentials

| Current | Action |
|---------|--------|
| WooCommerce API credentials (`vMZW4D7gnzSYVw0g`) | Replace with Shopify private app credentials (API key + access token) |
| WordPress App Password (`NF8QgSUEDF9pT6AB`) | Remove — no longer needed |

All other credentials remain unchanged:
- Postgres (`BSoGuZ9BOv4OWqXf`)
- Ollama (`mLc8L4ENefXAw6xJ`)
- Slack (`DvtYgxYgI1p5Q3QX`)
- Qdrant (API key)

### Server Infrastructure

No changes required to the Ubuntu server. All services (Supabase, n8n, Qdrant, Ollama, Crawl4AI) remain as-is.

---

## 9. Data Migration

### Products
- Export all WC products with ACF fields
- Map to Shopify product structure
- Create metafield definitions first
- Import products with metafields
- Preserve SKUs as the universal key (Leader `stock_code` → WC `sku` → Shopify `sku`)

### Categories → Collections
- WC hierarchical categories → Shopify smart collections (rule-based) or manual collections
- `tl_category_map` needs updating with Shopify collection IDs
- Consider smart collections using tags for automatic membership

### Attributes → Metafields
- Export WC global attribute definitions
- Create corresponding Shopify metafield definitions
- Migrate per-product attribute values to product metafields

### Customers and Orders
- Use Shopify's native import tools or a migration app (e.g., LitExtension, Cart2Cart)
- Order history important for returns/warranty
- Customer accounts need re-creation (passwords cannot be migrated)

### Images
- Shopify can sideload from existing URLs during import
- Original processed images (square + white padding) are on the WC media library — need URL references

### Redirects
- All WC product URLs need 301 redirects to Shopify equivalents
- Critical for SEO preservation

---

## 10. Cost Considerations

### Current (WooCommerce)
- **Hosting**: Self-managed (included in existing server costs)
- **WooCommerce**: Free (open source)
- **Theme/plugins**: One-time or annual licence costs
- **Payment gateway**: Direct integration (no platform fee)
- **Total platform cost**: Minimal — primarily hosting + domain

### Shopify
- **Basic plan**: ~$59 AUD/month
- **Shopify plan**: ~$159 AUD/month (recommended for the feature set)
- **Advanced plan**: ~$599 AUD/month
- **Transaction fees**: 0.5–2% if not using Shopify Payments (on top of gateway fees)
- **Shopify Payments**: No additional transaction fee, but Shopify Payments has its own processing rates
- **Apps**: Expect $20–100/month for apps replacing free WC plugins (structured specs, advanced SEO, etc.)
- **Theme**: $0–$400 one-time

### Hidden Costs
- Developer time to rebuild 5+ workflows (high-impact rewrites)
- Database schema migration and view updates
- Testing and validation period (running both platforms in parallel)
- Potential Shopify app costs for features that are free in WC (e.g., advanced custom fields)

---

## 11. What Stays the Same

These components have **zero WooCommerce dependency** and carry over unchanged:

- **TL_Ingest_Leader_Feed** — Leader XML ingestion to Supabase
- **Supabase core** — database, all non-WC tables, execution tracking
- **Qdrant** — vector database, RAG collections, embeddings
- **Ollama** — local LLM for title/description/attribute generation
- **Crawl4AI** — manufacturer website scraping
- **Slack integration** — channels, bot, Block Kit messages (only 3 button handlers need API swaps)
- **HITL approval flow** — Supabase-based queue and status management
- **Attribute alias resolution** — `tl_attribute_aliases` is platform-agnostic
- **LLM enrichment pipeline** — extraction from manufacturer data, not store data
- **Ubuntu server** — all services, Docker containers, networking

---

## Summary

| Area | Effort | Risk |
|------|--------|------|
| Core sync workflows (Mirror, Syncer, Publisher) | **High** | API differences, rate limiting, testing |
| Attribute system redesign | **High** | No direct Shopify equivalent of WC global attributes |
| Database schema migration | **Medium** | Mechanical but tedious — many views to update |
| Slack HITL adjustments | **Low** | Only 3 button handlers need API swaps |
| Enrichment pipeline | **Low** | Only the WC Pusher workflow needs changes |
| Data migration (products, customers, orders) | **Medium** | Tools exist but customer passwords can't migrate |
| SEO preservation (redirects) | **Medium** | Critical to get right, one-time effort |
| Ongoing costs | **Medium** | $100–300+/month vs near-zero platform cost currently |
| Leader feed + Supabase + AI stack | **None** | Completely unaffected |

The automation ecosystem is well-architected with Supabase as a decoupled middle layer, which significantly reduces migration risk. The hardest parts are the attribute system redesign (WC attributes → Shopify metafields) and rebuilding the 5 high-impact workflows. The enrichment pipeline, AI stack, and HITL flows are largely platform-agnostic by design.
