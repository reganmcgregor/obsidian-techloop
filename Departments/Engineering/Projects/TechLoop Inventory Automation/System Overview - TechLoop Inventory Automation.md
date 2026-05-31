# System Overview: TechLoop Inventory Automation

**Status:** 🟢 Production Active | ✅ Phases 1–6.2 Complete | 🚧 Phase 6.3 In Progress (7 of 9 enrichment workflows live)
**Project Lead:** Regan McGregor
**Last Updated:** 2026-02-28

---

## 1. Architecture Philosophy

The system follows a **"Raw over Modelled"** philosophy. Every field from the supplier feed is ingested and preserved in its raw state, alongside sanitized versions for operational use. This ensures zero data loss and allows for retrospective logic changes without re-ingesting historical data.

The system is **decoupled** into three distinct layers:
1.  **Ingestion:** Syncing the supplier feed to Supabase.
2.  **Mirroring:** Syncing the WooCommerce store state to Supabase.
3.  **Syncing:** Comparing the two states and pushing updates to WooCommerce.

---

## 2. Infrastructure

-   **Database:** Supabase PostgreSQL (Standard self-hosted via Docker).
    -   **Host:** `192.168.1.111`
    -   **URL:** `https://supabase.reganmcgregor.com.au`
-   **Automation:** n8n.
    -   **URL:** `https://n8n.reganmcgregor.com.au`
-   **Supplier:** Leader Computers (XML Feed).
-   **Store:** WooCommerce.

---

## 3. Workflow Catalog

### Workflow 1: Leader Ingest
-   **n8n Name:** `TL_Ingest_Leader_Feed`
-   **n8n ID:** `mUkhS0BfN7Lq6EsS`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/mUkhS0BfN7Lq6EsS)
-   **Frequency:** Every 2 hours at :00.
-   **Function:** Fetches the massive Leader XML feed, transforms tags to `snake_case`, sanitizes stock levels (handling "CALL"/"POA"), and performs a high-performance batch upsert into `tl_feed_leader_raw`.

### Workflow 2: WooCommerce Mirroring
-   **n8n Name:** `TL_Mirror_WooCommerce`
-   **n8n ID:** `hEUnN85kXJZEG3qZ`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/hEUnN85kXJZEG3qZ)
-   **Frequency:** Every 4 hours at :15.
-   **Function:** Performs an incremental sync of WooCommerce products into `tl_wc_products_mirror` using the `modified_after` filter. It extracts custom ACF fields (Supplier ID, DBP, MPN) from the metadata array into flat columns for fast querying.

### Workflow 3: Inventory Syncer (The Diff Engine)
-   **n8n Name:** `TL_Inventory_Syncer`
-   **n8n ID:** `ZgqlMhNqh3TJMB6G`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/ZgqlMhNqh3TJMB6G)
-   **Frequency:** Every 3 hours at :30.
-   **Function:** The core logic. Runs a complex SQL JOIN between the Leader feed and the WC Mirror. It identifies stock changes, cost (DBP) changes, and discontinued products (missing from feed for >2 hours). It pushes updates to WooCommerce and logs every change to `tl_sync_log`.

### Workflow 4: Price Watchdog
-   **n8n Name:** `TL_Price_Watchdog`
-   **n8n ID:** `yExNIbiIGVFOoJ4N`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/yExNIbiIGVFOoJ4N)
-   **Frequency:** Daily at 6:00 AM.
-   **Function:** Scans for low-margin products or items selling below cost. It generates an HTML email report with Price, RRP, and Cost comparisons, providing an early warning system for pricing errors.

### Workflow 5: Product Detector
-   **n8n Name:** `TL_Product_Detector`
-   **n8n ID:** `rnUjGGsdD1jdaFIg`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/rnUjGGsdD1jdaFIg)
-   **Frequency:** Every 2 hours at :45.
-   **Function:** Detects new products in the Leader feed that don't exist in WooCommerce. Queues them in `tl_onboarding_queue` for human review with category mapping. Auto-approves products in trusted categories. Sends Slack notifications to `#product-detector` with interactive HITL messages including Approve, Reject, and Google Search buttons for mobile-friendly product research.
-   **Documentation:** [[Workflows/Workflow 5 - Product Detector|Workflow 5 - Product Detector]]

### Workflow 6: Product Publisher
-   **n8n Name:** `TL_Product_Publisher`
-   **n8n ID:** `xBSqJjECViFMgUNI`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/xBSqJjECViFMgUNI)
-   **Frequency:** Every 5 minutes.
-   **Function:** Takes approved products from the queue and creates them in WooCommerce. Uses Ollama (`qwen2.5:14b`) for AI title optimisation (with Qdrant RAG examples), description cleanup (with `<h3>` headings), and short description generation (with Qdrant RAG examples). Processes product images (square with white padding) and uploads to WC Media Library. Products are created as drafts for final review. Creates a `tl_product_optimizations` record for each published product to track enrichment status.
-   **Documentation:** [[Workflows/Workflow 6 - Product Publisher|Workflow 6 - Product Publisher]]

### Workflow 7: Slack Interaction Handler (HITL Gateway)
-   **n8n Name:** `TL_Slack_Interaction_Handler`
-   **n8n ID:** `rDloLTYACT56kfpn`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/rDloLTYACT56kfpn)
-   **Trigger:** Webhook (`/webhook/slack-interaction`)
-   **Function:** Handles all Slack interactive callbacks (buttons and dropdowns) for HITL approval flows. Routes by `action_id` to 10 branches: approve/reject products, publish WC drafts, approve/reject/map-to-existing attributes, approve/reject terms, approve/reject mappings. Supports both `button` actions (`action.value`) and `static_select` dropdowns (`action.selected_option.value`). Updates the Slack message with confirmation after each action.
-   **Documentation:** See Section 5 (Operations & Monitoring) below for HITL channel and action mapping.

### Workflow 8: Queue Reviewer
-   **n8n Name:** `TL_Queue_Reviewer`
-   **n8n ID:** `I0eOOo8geZfhLKfs`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/I0eOOo8geZfhLKfs)
-   **Frequency:** Every 4 hours + Slack Slash Command (`/review-queue`).
-   **Function:** Periodically (or on-demand via `/review-queue` slash command) pulls a random set of 5 products from the `pending_review` queue and sends them to Slack for human review. Each product message includes Approve, Reject, and Google Search buttons for efficient mobile review workflow. Helps process the backlog of thousands of products in manageable chunks.
-   **Documentation:** [[Workflows/Workflow 8 - Queue Reviewer|Workflow 8 - Queue Reviewer]]

### Utility: Seed Qdrant Titles
-   **n8n Name:** `TL_Seed_Qdrant_Titles`
-   **n8n ID:** `WSZvDoxgUBmOuM8W`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/WSZvDoxgUBmOuM8W)
-   **Frequency:** Manual (one-time).
-   **Function:** Populates the Qdrant `product_titles` collection with existing good product titles from `tl_wc_products_mirror` for use as RAG examples by the Product Publisher.

### Utility: Seed Qdrant Short Descriptions
-   **n8n Name:** `TL_Seed_Qdrant_Short_Descriptions`
-   **n8n ID:** `2M95azR2qs49Tku4`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/2M95azR2qs49Tku4)
-   **Frequency:** Manual (one-time).
-   **Function:** Populates the Qdrant `product_short_descriptions` collection with existing short descriptions from `tl_wc_products_mirror` for use as RAG examples by the Product Publisher.

### Utility: Import WC Brands
-   **n8n Name:** `TL_Import_WC_Brands (Utility)`
-   **n8n ID:** `I1xF1z82hG423xP2`
-   **Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/I1xF1z82hG423xP2)
-   **Frequency:** Webhook-triggered.
-   **Function:** Imports WooCommerce brand taxonomy terms into `tl_brand_map` for brand lookup during publishing.

---

## 4. Key Database Views

### Core Views (Phase 1-4)
-   `vw_products_needs_sync`: Identifies products where Supabase detects a mismatch between supplier and store.
-   `vw_low_margin_products`: Core data source for the Price Watchdog alert system.
-   `vw_discontinued_products`: Lists products no longer appearing in the supplier feed.

### Product Automation Views (Phase 6)
-   `vw_new_products_for_onboarding`: Core view for Product Detector - shows products eligible for onboarding.
-   `vw_onboarding_queue_summary`: Dashboard view showing queue counts by status.
-   `vw_unmapped_categories`: Shows Leader categories without WooCommerce mapping.
-   `vw_products_need_description`: Products pending AI description optimisation (Phase 6.3).

---

## 5. Operations & Monitoring

-   **Audit Trail:** Every single change made to WooCommerce is logged in `tl_sync_log`.
-   **Workflow Health:** Monitored via the `tl_workflow_executions` table.
-   **Notifications:** All workflow notifications sent to Slack (migrated from email 2026-02-07). Channels: `#product-detector`, `#product-publisher`, `#price-watchdog`, `#enrichment-review`.
-   **HITL Approvals:** Approve/reject buttons in `#product-detector`, publish/edit/preview buttons in `#product-publisher`. Handled by `TL_Slack_Interaction_Handler` webhook workflow.
-   **Alerts:** Price alerts sent to `#price-watchdog` daily. Critical workflow failures handled by n8n error catchers.

### Monitoring Queries

#### Check Latest Executions
```sql
SELECT workflow_name, started_at, status, duration_seconds, records_processed
FROM tl_workflow_executions
ORDER BY started_at DESC
LIMIT 10;
```

#### Check Recent Sync Changes
```sql
SELECT sku, change_type, field_changed, old_value, new_value, created_at
FROM tl_sync_log
ORDER BY created_at DESC
LIMIT 20;
```

---

## 6. Phase 6: Product Automation (Complete)

Phase 6 extends the system to handle **new product onboarding** from the Leader feed. Phase 6.1 (Detector) and 6.2 (Publisher) are complete and in production.

### New Database Tables
-   `tl_category_map`: Maps Leader categories to WooCommerce categories with blocking/auto-approve settings.
-   `tl_brand_map`: Maps supplier brand names to WooCommerce brand taxonomy terms.
-   `tl_onboarding_queue`: Staging queue for products awaiting review and publishing.
-   `tl_product_optimizations`: Tracks enrichment status for published products (description, images, attributes). Each field defaults to `pending` — the next workflow (Phase 6.3) will query these to find products needing enrichment.

### Key Features
-   **Category Mapping:** Control which products can be onboarded via `tl_category_map`.
-   **Brand Mapping:** Auto-lookup and auto-create brands via `tl_brand_map`.
-   **Human Review:** Products queue for approval unless auto-approved by category.
-   **AI Title Optimisation:** Ollama (`qwen2.5:14b`) + Qdrant RAG for SEO-friendly titles.
-   **AI Description Cleanup:** Ollama reformats raw specs into clean HTML with `<h3>` headings.
-   **AI Short Description:** Ollama + Qdrant RAG generates bullet-point short descriptions.
-   **Image Processing:** Square with white padding, uploaded to WC Media Library.
-   **Draft Publishing:** Products created as drafts for final review.

### Documentation
-   [[3-Resources/Infrastructure/Database Schema - Product Automation|Database Schema - Product Automation]]
-   [[Workflows/Workflow 5 - Product Detector|Workflow 5 - Product Detector]]
-   [[Workflows/Workflow 6 - Product Publisher|Workflow 6 - Product Publisher]]
-   [[Config/Category Mapping Guide|Category Mapping Guide]]

---

## 7. Future Roadmap

### Phase 5 (Maintenance)
-   **Load Testing:** Verify performance with 10,000+ products.
-   **Runbooks:** Detailed troubleshooting guides for each workflow.
-   **Retention:** Automated cleanup of logs older than 90 days.

### Phase 6.3 (Product Enrichment Suite - In Progress)

Full enrichment pipeline with manufacturer data scraping, structured attribute extraction, feature-focused descriptions, and image sourcing.

**Live Workflows (7 of 9):**
-   `TL_URL_Discoverer` (ID: `QQMwVzR5c79Nx3RL`) - Finds manufacturer URLs from Leader feed data, bulk-updates `tl_product_optimizations` (daily at 02:00 AEST)
-   `TL_Scraper` (ID: `b4rR1gwQ25uUJkEQ`) - Fetches manufacturer pages via Crawl4AI (Playwright/stealth), stores `markdown_content` + `scraped_images` in `tl_manufacturer_raw_data` (every 2 hours, 5 products/run)
-   `TL_Mirror_WC_Attributes` (ID: `UhdbMqPKwfMV4BSW`) - Mirrors WC attribute definitions + terms to Supabase (daily at 02:00 AEST)
-   `TL_Enrich_Attributes` (ID: `lq8960K0xVwF3Xst`) - LLM attribute extraction using Ollama `qwen2.5:14b` from `markdown_content` (every 15 min, 5 products/run)
-   `TL_Attribute_Proposer` (ID: `hUGA1KFWBTGU8K6T`) - Groups unknown attrs/terms, sends batch HITL to `#enrichment-review` (daily at 03:00 AEST)
-   `TL_Enrichment_Reviewer` (ID: `OarmHP6DpJvAV0Pb`) - Sends pending attribute mappings to Slack for review; deduplicates via `sent_to_slack_at` (every 4h)
-   `TL_Attribute_WC_Pusher` (ID: `EOySGcLHT8Mv62DU`) - Pushes approved attribute mappings to WC products via API (every 10 min)

**Planned Workflows (2 of 9):**
-   `TL_Enrich_Descriptions` - Feature-focused descriptions from `markdown_content` (every 15 min)
-   `TL_Enrich_Images` - Processes `scraped_images` JSONB from data lake, uploads to WC gallery (every 30 min)

**New Database Tables:** `tl_manufacturer_raw_data`, `tl_wc_attributes_mirror`, `tl_wc_attribute_terms_mirror`, `tl_proposed_attributes`, `tl_product_attributes`, `tl_description_models`, `tl_attribute_aliases`

See [[Phase 6.3 - Product Enrichment Pipeline]] for full specification.
See [[Future Roadmap - TechLoop Inventory Automation|Future Roadmap]] for context.