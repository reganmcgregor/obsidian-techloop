# Database Schema: Product Automation (Phase 6)

**Status:** Production (Phase 6.1-6.2), Phase 6.3 In Progress
**Created:** 2026-02-01
**Last Updated:** 2026-06-24
**SQL File:** [[Database Schema - Product Automation.sql]]

> **Reconciled 2026-06-24** against the live Supabase schema after the Phase 13 WooCommerce decommission — dropped WC mirror tables removed; `wc_product_id`→`shopify_product_id`, `wc_product_url`→`product_url`, `mapped_wc_category_*`→`mapped_category_*`; views repointed to `tl_shopify_products_mirror`. The `wc_attribute_id` / `wc_slug` attribute keys are retained as live lookup keys (owned by the Product Enrichment Engine).

---

## Overview

Phase 6 extends the TechLoop Inventory Automation database to support new product onboarding (6.1-6.2) and the full enrichment pipeline (6.3). Phase 6.3 adds manufacturer data scraping, structured attribute extraction, configurable description generation, and HITL review workflows.

---

## Tables

### 1. tl_category_map

Maps Leader categories to target categories with automation controls (gating, auto-approve, margin). Shopify taxonomy + collection mapping lives in `tl_shopify_category_map`. *(2026-06-24: `wc_category_id`/`wc_category_name` renamed to `mapped_category_id`/`mapped_category_name` — see `wc-naming-cleanup-2026-06-24.sql`.)*

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `supplier_code` | VARCHAR(50) | Supplier identifier (default: 'LEADER') |
| `supplier_category_code` | VARCHAR(100) | Leader category code (e.g., 'HARDWARE > NOTEBOOKS') |
| `supplier_category_name` | VARCHAR(255) | Human-readable category name |
| `mapped_category_id` | INTEGER | Mapped category id (legacy WooCommerce term_id) |
| `mapped_category_name` | VARCHAR(255) | Mapped category name (for reference) |
| `is_blocked` | BOOLEAN | If true, products in this category are auto-rejected |
| `block_reason` | TEXT | Reason for blocking (e.g., 'Low margin category') |
| `auto_approve` | BOOLEAN | If true, products skip review and publish automatically |
| `default_margin_percent` | NUMERIC(5,2) | Default markup percentage (default: 20%) |

**Key Indexes:**
- `idx_tl_category_map_lookup` - Fast lookup for non-blocked categories
- `idx_tl_category_map_unmapped` - Find categories without a mapped category

**Usage:**
- Manually populate this table with category mappings before products can be published
- Use `is_blocked = true` to prevent entire categories from being onboarded
- Use `auto_approve = true` for trusted categories (e.g., laptops)

---

### 2. tl_onboarding_queue

Staging queue for human review before products enter WooCommerce.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `stock_code` | VARCHAR(100) | Leader stock code (unique) |
| `product_name` | VARCHAR(500) | Short product name |
| `product_name_2` | TEXT | Long description (may contain HTML) |
| `manufacturer` | VARCHAR(255) | Brand name |
| `manufacturer_sku` | VARCHAR(100) | MPN (Manufacturer Part Number) |
| `bar_code` | VARCHAR(100) | Barcode/EAN/GTIN |
| `category_code` | VARCHAR(100) | Leader category code |
| `category_name` | VARCHAR(255) | Leader category name |
| `subcategory_name` | VARCHAR(255) | Leader subcategory |
| `dbp_ex_gst` | NUMERIC(12,2) | Dealer Buy Price (Ex GST) |
| `dbp_inc_gst` | NUMERIC(12,2) | Dealer Buy Price (Inc GST) |
| `rrp` | NUMERIC(12,2) | Recommended Retail Price |
| `image_url` | TEXT | Product image URL |
| `availability_total` | INTEGER | Stock level at detection |
| `weight` | VARCHAR(50) | Product weight |
| `length` | VARCHAR(50) | Product length |
| `width` | VARCHAR(50) | Product width |
| `height` | VARCHAR(50) | Product height |
| `warranty_length` | VARCHAR(100) | Warranty period |
| `mapped_wc_category_id` | INTEGER | WooCommerce category from mapping |
| `mapped_wc_category_name` | VARCHAR(255) | WooCommerce category name |
| `status` | VARCHAR(50) | Queue status (see below) |
| `priority` | INTEGER | Publishing priority (1-100, higher first) |
| `rejection_reason` | TEXT | Why product was rejected |
| `processing_started_at` | TIMESTAMPTZ | Lock for preventing race conditions |
| `retry_count` | INTEGER | Number of publish attempts |
| `detected_at` | TIMESTAMPTZ | When product was first detected |
| `reviewed_at` | TIMESTAMPTZ | When product was reviewed |
| `reviewed_by` | VARCHAR(100) | Who reviewed the product |
| `published_at` | TIMESTAMPTZ | When product was published |
| `wc_product_id` | INTEGER | WooCommerce product ID after creation |
| `wc_product_url` | TEXT | URL to product in WooCommerce |
| `publish_error` | TEXT | Error message if publishing failed |
| `raw_data` | JSONB | Full Leader feed data backup |

**Status Values:**
| Status | Description |
|--------|-------------|
| `pending_review` | Awaiting human review |
| `approved` | Ready for publishing |
| `rejected` | Will not be published |
| `blocked` | Category is blocked |
| `processing` | Currently being published |
| `published` | Successfully created in WooCommerce |
| `failed` | Publishing failed (check `publish_error`) |

**Key Indexes:**
- `idx_tl_onboarding_queue_status` - Find products by status
- `idx_tl_onboarding_queue_approved` - Fetch approved products in priority order
- `idx_tl_onboarding_queue_pending` - Find products awaiting review
- `idx_tl_onboarding_queue_failed` - Find failed products for retry

---

### 3. tl_product_optimizations

Tracks optimisation status for published products. Central tracking table for all enrichment workflows.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `wc_product_id` | INTEGER | WooCommerce product ID (unique) |
| `stock_code` | VARCHAR(100) | Leader stock code |
| `sku` | VARCHAR(100) | WooCommerce SKU |
| `original_raw_title` | VARCHAR(500) | Raw title from Leader |
| `published_title` | VARCHAR(500) | Cleaned title we published |
| `published_slug` | VARCHAR(500) | URL slug from WooCommerce |
| `description_status` | VARCHAR(50) | Description optimisation status |
| `original_description` | TEXT | Raw description from Leader |
| `optimized_description` | TEXT | AI-generated description |
| `description_processed_at` | TIMESTAMPTZ | When description was optimised |
| `description_error` | TEXT | Error if description failed |
| `image_status` | VARCHAR(50) | Image optimisation status |
| `image_sources` | JSONB | URLs of scraped images |
| `image_processed_at` | TIMESTAMPTZ | When images were processed |
| `attribute_status` | VARCHAR(50) | Attribute extraction status |
| `extracted_attributes` | JSONB | Parsed specs from description |
| `attribute_processed_at` | TIMESTAMPTZ | When attributes were extracted |
| `manufacturer_url` | TEXT | Manufacturer product page URL *(Phase 6.3)* |
| `manufacturer_url_source` | VARCHAR(50) | How URL was found: leader_feed, serp_discovery, manual *(Phase 6.3)* |
| `scrape_status` | VARCHAR(50) | Scraping status: pending, completed, failed, skipped *(Phase 6.3)* |
| `description_version` | INTEGER | Increments when description regenerated *(Phase 6.3)* |
| `description_model_id` | UUID FK | Which LLM model generated the description *(Phase 6.3)* |
| `image_error` | TEXT | Error message from image processing *(Phase 6.3)* |

**Optimisation Statuses:**
- `pending` - Not yet processed
- `processing` - Currently being processed
- `completed` - Successfully optimised
- `failed` - Optimisation failed
- `skipped` - Skipped (e.g., no description needed)

---

## Views

### vw_new_products_for_onboarding

**Purpose:** Core view for `TL_Product_Detector` - identifies products eligible for onboarding.

**Logic:**
- Product must be seen in feed within last 4 hours
- Product must have stock (`availability_total > 0`)
- Product must have a price (`dbp_inc_gst > 0`)
- Product must NOT exist in WooCommerce (`tl_wc_products_mirror`)
- Product must NOT already be in the queue (`tl_onboarding_queue`)
- Category must NOT be blocked

**Columns:** All product fields from `tl_feed_leader_raw` plus category mapping fields.

---

### vw_onboarding_queue_summary

**Purpose:** Dashboard view showing queue counts by status.

**Columns:**
- `status` - Queue status
- `count` - Number of products
- `oldest` - Oldest detection date
- `newest` - Newest detection date

---

### vw_onboarding_queue_recent

**Purpose:** Shows recent queue activity (last 7 days).

**Columns:** Key product fields and queue status for recent items.

---

### vw_unmapped_categories

**Purpose:** Identifies Leader categories without WooCommerce mapping.

**Columns:**
- `category_code` - Leader category code
- `category_name` - Leader category name
- `product_count` - Total products in category
- `in_stock_count` - Products with stock

**Usage:** Use this view to identify categories that need mapping before products can be published.

---

### vw_products_need_description

**Purpose:** Products pending description optimisation (Phase 6.3).

**Columns:** Product details for items with `description_status = 'pending'`.

---

## Entity Relationship

```
tl_feed_leader_raw (existing)
        |
        | LEFT JOIN on category_code
        v
tl_category_map ──────────────────────────────┐
        |                                      |
        | provides mapped_category_id          |
        v                                      |
vw_new_products_for_onboarding                 |
        |                                      |
        | TL_Product_Detector inserts          |
        v                                      |
tl_onboarding_queue                            |
        |                                      |
        | TL_Product_Publisher creates product |
        v                                      |
tl_wc_products_mirror (existing) <─────────────┘
        |
        | Creates record at publish time
        v
tl_product_optimizations
```

---

## Common Operations

### Approve a Product

```sql
UPDATE tl_onboarding_queue
SET status = 'approved',
    reviewed_at = NOW(),
    reviewed_by = 'regan@techloop.com.au'
WHERE stock_code = 'ABC123';
```

### Reject a Product

```sql
UPDATE tl_onboarding_queue
SET status = 'rejected',
    rejection_reason = 'Not suitable for our store',
    reviewed_at = NOW(),
    reviewed_by = 'regan@techloop.com.au'
WHERE stock_code = 'ABC123';
```

### Block a Category

```sql
UPDATE tl_category_map
SET is_blocked = true,
    block_reason = 'Low margin category'
WHERE supplier_category_code = 'ACCESSORIES > CABLES';
```

### Add a Category Mapping

```sql
INSERT INTO tl_category_map (
    supplier_code,
    supplier_category_code,
    supplier_category_name,
    mapped_category_id,
    mapped_category_name,
    auto_approve,
    default_margin_percent
) VALUES (
    'LEADER',
    'HARDWARE > NOTEBOOKS',
    'Notebooks',
    155,
    'Laptops',
    false,
    18.00
);
```

### Check Queue Status

```sql
SELECT * FROM vw_onboarding_queue_summary;
```

### Find Unmapped Categories

```sql
SELECT * FROM vw_unmapped_categories;
```

---

---

## Phase 6.3 Tables (Product Enrichment Pipeline)

### 4. tl_manufacturer_raw_data

Scraped manufacturer page content. Scrape once, parse many times for future workflows (reviews, videos, FAQs). Supports multiple rows per product via `UNIQUE(stock_code, source_url)` — designed for multi-page scraping (e.g. main page + `/sp` specs + `/gallery`).

**Scraper:** Crawl4AI (Playwright-based, stealth mode) at `192.168.1.111:11235`. Replaced Browserless 2026-02-28.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `stock_code` | VARCHAR(100) | Links to `tl_feed_leader_raw` |
| `source_url` | TEXT | Manufacturer page URL |
| `url_source` | VARCHAR(50) | `leader_feed` / `serp_discovery` / `manual` |
| `raw_html` | TEXT | Full page HTML (archival) |
| `markdown_content` | TEXT | Clean Markdown from Crawl4AI — primary source for all LLM enrichment |
| `scraped_images` | JSONB | `[{src, alt, tier, confidence, format}]` extracted by Crawl4AI |
| `scraped_videos` | JSONB | `[{src, type, title}]` |
| `scrape_status` | VARCHAR(50) | `pending` / `completed` / `failed` / `skipped` |
| `scrape_method` | VARCHAR(50) | `crawl4ai` |
| `retry_count` | INTEGER | Default 0 |

**UNIQUE:** `(stock_code, source_url)`

**Dropped 2026-02-28** (redundant with Crawl4AI Markdown): `raw_text`, `spec_tables_html`, `feature_sections_html`

---

### 5. tl_wc_attributes_mirror

Mirror of WooCommerce global attribute definitions (`GET /products/attributes`).

| Column | Type | Description |
|--------|------|-------------|
| `wc_attribute_id` | INTEGER PK | From WooCommerce |
| `name` | VARCHAR(255) | e.g., "RAM", "Screen Size" |
| `slug` | VARCHAR(255) UNIQUE | |
| `type` | VARCHAR(50) | `select` |
| `order_by` | VARCHAR(50) | |
| `has_archives` | BOOLEAN | |
| `category_scope` | TEXT[] | Which WC categories this applies to (local metadata) |
| `unit` | VARCHAR(50) | e.g., "GB", "GHz", "inch" (local metadata) |
| `last_synced_at` | TIMESTAMPTZ | |

---

### 6. tl_wc_attribute_terms_mirror

Mirror of WooCommerce attribute terms (`GET /products/attributes/<id>/terms`).

| Column | Type | Description |
|--------|------|-------------|
| `wc_term_id` | INTEGER PK | From WooCommerce |
| `wc_attribute_id` | INTEGER FK | → `tl_wc_attributes_mirror` |
| `name` | VARCHAR(255) | e.g., "16GB", "14 inch" |
| `slug` | VARCHAR(255) | |
| `count` | INTEGER | Products using this term in WC |
| `last_synced_at` | TIMESTAMPTZ | |

**UNIQUE:** `(wc_attribute_id, slug)`

---

### 7. tl_description_models

Configurable LLM models for description generation. Supports Ollama, OpenRouter, Anthropic.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `provider` | VARCHAR(50) | `ollama` / `openrouter` / `anthropic` |
| `model_name` | VARCHAR(255) | e.g., `qwen2.5:14b`, `google/gemini-2.0-flash` |
| `api_endpoint` | TEXT | API URL |
| `api_key_credential_id` | VARCHAR(100) | n8n credential ID (NULL for Ollama) |
| `is_default` | BOOLEAN | |
| `cost_per_1k_tokens` | NUMERIC | For spend tracking |
| `status` | VARCHAR(50) | `active` / `disabled` |

---

### 8. tl_proposed_attributes

LLM-proposed new attributes or terms awaiting HITL approval via Slack.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `proposal_type` | VARCHAR(50) | `new_attribute` / `new_term` |
| `attribute_name` | VARCHAR(255) | For new attributes |
| `wc_attribute_id` | INTEGER FK | For new terms (which attribute) |
| `term_value` | VARCHAR(255) | For new terms |
| `product_count` | INTEGER | How many products need this |
| `example_products` | JSONB | Sample stock_codes |
| `status` | VARCHAR(50) | `proposed` / `approved` / `rejected` / `mapped` |

---

### 9. tl_attribute_aliases

Maps alternative attribute names (from manufacturer specs, supplier data, or HITL corrections) to canonical WC attributes. Used by `TL_Enrich_Attributes` to resolve known synonyms before proposing new attributes.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | Default `gen_random_uuid()` |
| `alias_name` | VARCHAR(255) NOT NULL | The alternative name (e.g., "Memory", "System Memory") |
| `maps_to_wc_attribute_id` | INTEGER NOT NULL FK | → `tl_wc_attributes_mirror` (the canonical attribute) |
| `source` | VARCHAR(50) | How the alias was created: `hitl_mapping` / `hitl_rejection` / `manual` |
| `created_at` | TIMESTAMPTZ | Default `NOW()` |

**UNIQUE:** `(alias_name)` — each alias maps to exactly one canonical attribute.

**How aliases are created:**
- **HITL mapping:** When `TL_Attribute_Proposer` shows a proposed new attribute in Slack, the reviewer can select "Map to Existing" from a dropdown instead of creating a new WC attribute. This inserts an alias.
- **Manual:** Direct DB insert for known synonyms (e.g., "M/B Type" → "Motherboard Form Factor").

**How aliases are consumed:**
- `TL_Enrich_Attributes` fetches aliases via LEFT JOIN on the attribute dictionary query. Aliases appear in the LLM prompt as "(also known as: Memory, System Memory)" next to each attribute.
- The Parse LLM Response node also builds a runtime alias lookup — if the LLM returns an alias name despite being told to use the canonical name, it resolves correctly instead of proposing a new attribute.

---

### 10. tl_product_attributes

Per-product attribute extraction results. Maps products to WC attributes with confidence scores.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `wc_product_id` | INTEGER | WooCommerce product ID |
| `stock_code` | VARCHAR(100) | Leader stock code |
| `wc_attribute_id` | INTEGER FK | → `tl_wc_attributes_mirror` |
| `raw_value` | TEXT | As extracted by LLM |
| `normalized_value` | TEXT | Matched to existing term |
| `wc_term_id` | INTEGER | NULL until matched to WC term |
| `confidence` | NUMERIC(3,2) | 0.00–1.00 |
| `source` | VARCHAR(50) | `llm_extraction` / `leader_feed` / `manual` |
| `extraction_run` | INTEGER | Increments on re-extraction |
| `review_status` | VARCHAR(50) | `pending` / `approved` / `rejected` / `pushed_to_wc` |
| `pushed_to_wc_at` | TIMESTAMPTZ | |
| `sent_to_slack_at` | TIMESTAMPTZ | When notification was sent to `#enrichment-review` *(Added 2026-02-21)* |
| `slack_notification_count` | INTEGER | Number of times sent to Slack (default 0) *(Added 2026-02-21)* |

**UNIQUE:** `(wc_product_id, wc_attribute_id, normalized_value)` *(Updated 2026-02-22 — allows multiple values per attribute per product; old constraint was `(wc_product_id, wc_attribute_id)`)*

**Notification Tracking (Added 2026-02-21):**
- `sent_to_slack_at` tracks when the item was last sent to Slack for review
- `slack_notification_count` increments each time a notification is sent
- `TL_Enrichment_Reviewer` only sends items where `sent_to_slack_at IS NULL` OR `sent_to_slack_at < NOW() - INTERVAL '1 day'`
- This prevents duplicate Slack messages every 4 hours for the same pending item

---

### Modified Existing Table: tl_wc_products_mirror

Added `attributes` JSONB column — mirrors WC product attributes array: `[{id, name, slug, position, visible, variation, options}]`. Synced by `TL_Mirror_WooCommerce`.

---

## Phase 6.3 Views

### vw_products_need_scraping
Products with manufacturer URLs ready for scraping (excludes products with 3+ failures).

### vw_products_need_url_discovery
Products without a manufacturer URL — candidates for URL discovery via SERP.

### vw_products_need_attribute_extraction
Products with `attribute_status = 'pending'`. Aggregates `markdown_content` via `STRING_AGG` across all completed rows in `tl_manufacturer_raw_data` for each product — supports future multi-URL scraping without view changes. Prioritises products with manufacturer data over those relying only on the Leader feed description.

### vw_proposed_attributes
Pending HITL proposals — new attributes and new terms grouped by product count.

### vw_pending_attribute_mappings
Product-attribute values awaiting review — high confidence first for auto-approval.

### vw_attribute_coverage
Per-attribute breakdown of how many tracked products have each attribute filled (quality metric).

---

## Entity Relationship (Phase 6.3)

```
tl_wc_attributes_mirror ←── TL_Mirror_WC_Attributes ←── WooCommerce /products/attributes
        |
        ├── FK
        |   v
        |  tl_wc_attribute_terms_mirror ←── WC /products/attributes/<id>/terms
        |
        ├── FK (maps_to_wc_attribute_id)
        |   v
        |  tl_attribute_aliases ←── HITL "Map to Existing" / manual inserts
        |
        ├── FK (wc_attribute_id)
        |   v
        |  tl_product_attributes ←── TL_Enrich_Attributes (LLM extraction)
        |       UNIQUE(wc_product_id, wc_attribute_id, normalized_value)
        |       review_status: pending → approved → pushed_to_wc
        |
        └── FK (wc_attribute_id)
            v
           tl_proposed_attributes ←── TL_Attribute_Proposer (HITL via Slack)
                status: proposed → approved/rejected/mapped

tl_manufacturer_raw_data ←── TL_Scraper (Crawl4AI — Playwright, stealth mode)
        |  UNIQUE(stock_code, source_url) — multiple rows per product supported
        |
        | markdown_content → feeds TL_Enrich_Attributes + TL_Enrich_Descriptions
        | scraped_images   → feeds TL_Enrich_Images
        | raw_html         → archival (scrape once, parse many times)

tl_description_models ←── configurable LLM selection for descriptions
```

---

## Related Documentation

- [[System Overview - TechLoop Inventory Automation]]
- [[Phase 6.3 - Product Enrichment Pipeline]]
- [[Workflow 5 - Product Detector]]
- [[Workflow 6 - Product Publisher]]
- [[Category Mapping Guide]]
- [[Database Schema - TechLoop Inventory]]
