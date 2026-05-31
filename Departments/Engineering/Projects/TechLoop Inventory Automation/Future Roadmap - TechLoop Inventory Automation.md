# Future Roadmap: TechLoop Inventory Automation

This document outlines the remaining tasks and long-term roadmap for the inventory automation system as of **2026-02-28**.

---

## 1. 📊 Monitoring Dashboards (TEC-15)
**Status:** 🚧 Partial (Built into System Overview)
**Next Steps:**
- Create a dedicated Obsidian dashboard note using the `Dataview` or `Tasks` plugin to visualize `tl_workflow_executions`.
- Integrate daily sync summaries into the "Daily Note" template.

## 2. 🧪 Production Load Testing (TEC-17)
**Status:** ⏳ Pending
**Objective:** Verify system stability with the full 10,000+ item catalog.
**Plan:**
- Monitor the duration of `TL_Ingest_Leader_Feed` over the next 7 days.
- Check memory usage on the Ubuntu server (`192.168.1.111`) during peak ingest.
- Optimize Supabase indexes if diff query duration exceeds 30 seconds.

## 3. 📖 Operations Runbooks (TEC-18)
**Status:** 🚧 Started (Consolidated in System Overview)
**Next Steps:**
- Document specific recovery steps for `429 Too Many Requests` from WooCommerce API.
- Create a "Discontinuation Manual Override" guide for items that should remain active even if missing from the feed.

## 4. 🧹 Data Retention & Cleanup (TEC-19)
**Status:** ⏳ Pending
**Objective:** Prevent database bloat in Supabase.
**Plan:**
- Build a simple n8n cleanup workflow that runs weekly.
- **Rules:**
    - `tl_sync_log`: Keep 90 days of history.
    - `tl_workflow_executions`: Keep 30 days of history.
- Implement a `pg_cron` job or n8n node to execute `DELETE` queries based on these rules.

---

## 5. ✅ Phase 6.1-6.2: Product Automation (Complete)
**Status:** ✅ Production (2026-02-07)

Phase 6 extends the system to handle new product onboarding from the Leader feed.

### Phase 6.1: Database + Product Detector
-   **Tables:** `tl_category_map`, `tl_brand_map`, `tl_onboarding_queue`, `tl_product_optimizations`
-   **Views:** `vw_new_products_for_onboarding`, `vw_onboarding_queue_summary`, `vw_unmapped_categories`
-   **Workflow:** `TL_Product_Detector` - Detects new products, queues for review
-   **Documentation:** [[3-Resources/Infrastructure/Database Schema - Product Automation|Database Schema - Product Automation]], [[Workflows/Workflow 5 - Product Detector|Workflow 5 - Product Detector]]

### Phase 6.2: Product Publisher
-   **Workflow:** `TL_Product_Publisher` - Publishes approved products to WooCommerce (every 5 min)
-   **AI Features:** Title optimisation (Ollama + Qdrant RAG), description cleanup (`<h3>` headings), short description generation (Qdrant RAG)
-   **Image Processing:** Square with white padding, uploaded to WC Media Library
-   **Model:** `qwen2.5:14b` for all AI generation, `nomic-embed-text` for embeddings
-   **Tracking:** Creates `tl_product_optimizations` record per product (all statuses default to `pending`)
-   **Documentation:** [[Workflows/Workflow 6 - Product Publisher|Workflow 6 - Product Publisher]], [[Config/Category Mapping Guide|Category Mapping Guide]]

---

## 6. 📝 Phase 6.3: Product Enrichment Suite (In Progress)
**Status:** 🚧 In Progress (started 2026-02-15)
**Prerequisite:** Phase 6.1-6.2 ✅ Complete
**Full Specification:** [[Phase 6.3 - Product Enrichment Pipeline]]

Phase 6.3 is a full enrichment pipeline with four phases:

| Phase | Purpose | Status |
|-------|---------|--------|
| **Phase 0:** Scraping | Manufacturer data lake — Crawl4AI (Playwright/stealth), `markdown_content` + `scraped_images` | ✅ Live |
| **Phase 1:** Attributes | Structured attribute extraction with WC mirror, HITL for unknowns, WC push | ✅ Live |
| **Phase 2:** Descriptions | Feature-focused descriptions (not specs), configurable LLM models | ⏳ Planned |
| **Phase 3:** Images | Process `scraped_images` from data lake, upload to WC gallery | ⏳ Planned |

### Key Design Decisions
- **Attribute system separated by WC API concern:** Mirror (definitions/terms), Extraction (LLM), HITL (new attrs/terms), Push (product assignment)
- **Re-runnable extraction:** New data sources auto-reset `attribute_status` → re-extraction with richer context
- **Descriptions = features, not specs:** Specs handled by Attributes tab. Descriptions focus on benefits and use cases.
- **Scrape once, parse many:** Full page HTML stored for future workflows (reviews, videos)
- **Markdown as primary LLM source:** Crawl4AI provides clean Markdown — used directly by enrichment workflows instead of raw HTML. Replaced Browserless 2026-02-28.
- **Brand TOV:** [[Tone of Voice]] and [[AI Writing Guidelines]] referenced in all LLM prompts

### Future Scope — Multi-Page Product Scraping

Many manufacturer sites split product information across multiple URLs. Example (Gigabyte):

- `.../B850M-EAGLE-WIFI6E-ICE-rev-10` — features/overview → best for **Descriptions**
- `.../B850M-EAGLE-WIFI6E-ICE-rev-10/sp` — full spec tables → best for **Attributes**
- `.../B850M-EAGLE-WIFI6E-ICE-rev-10/gallery` — image gallery → best for **Images**

**Schema is already ready:** `tl_manufacturer_raw_data` uses `UNIQUE(stock_code, source_url)`, so a product can have multiple rows — one per URL scraped. `vw_products_need_attribute_extraction` already aggregates `markdown_content` across all rows per product via `STRING_AGG`.

**What needs building:**
- `TL_URL_Discoverer` enhanced to detect sub-page URLs (`/sp`, `/gallery`, `/specifications`) from the main page HTML and queue them as additional scrape targets
- `tl_product_optimizations.scrape_status` becomes a derived rollup: `completed` only when all known URLs for a product are `completed` in `tl_manufacturer_raw_data`

**Image note:** Crawl4AI captures `<img>` tags, `<picture>`, and `srcset` (including lazy-loaded after `scan_full_page`), but not CSS `background-image`. For tech product pages this is acceptable — product images are almost always in `<img>` tags. Revisit if high-value sites are found to use CSS backgrounds for product photos.

### New Workflows (9 total)
`TL_Mirror_WC_Attributes`, `TL_Enrich_Attributes`, `TL_Attribute_Proposer`, `TL_Attribute_WC_Pusher`, `TL_Scraper`, `TL_URL_Discoverer`, `TL_Enrich_Descriptions`, `TL_Enrich_Images`, `TL_Enrichment_Reviewer`

### New Tables (6 total)
`tl_manufacturer_raw_data`, `tl_wc_attributes_mirror`, `tl_wc_attribute_terms_mirror`, `tl_proposed_attributes`, `tl_product_attributes`, `tl_description_models`

---

## 7. 🚀 Beyond Phase 6
- **Multi-Supplier Support:** Extend the `tl_feed_leader_raw` pattern to other suppliers (e.g., Ingram Micro).
- **Price Automation:** Automatically calculate and push new retail prices based on margin rules, rather than just monitoring.
- **Frontend Integration:** Build a simple internal tool to view the "Mirror vs Feed" state directly in the browser.
- **Competitor Price Monitoring:** Track competitor pricing for key products.
- **Demand Forecasting:** Use sales history to predict optimal stock levels.
