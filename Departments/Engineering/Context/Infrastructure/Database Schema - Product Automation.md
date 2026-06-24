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

Staging queue for human review before products are published to Shopify.

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
| `mapped_category_id` | INTEGER | Mapped target category id (from `tl_category_map`) |
| `mapped_category_name` | VARCHAR(255) | Mapped target category name |
| `status` | VARCHAR(50) | Queue status (see below) |
| `priority` | INTEGER | Publishing priority (1-100, higher first) |
| `rejection_reason` | TEXT | Why product was rejected |
| `processing_started_at` | TIMESTAMPTZ | Lock for preventing race conditions |
| `retry_count` | INTEGER | Number of publish attempts |
| `detected_at` | TIMESTAMPTZ | When product was first detected |
| `reviewed_at` | TIMESTAMPTZ | When product was reviewed |
| `reviewed_by` | VARCHAR(100) | Who reviewed the product |
| `published_at` | TIMESTAMPTZ | When product was published |
| `shopify_product_id` | INTEGER | Shopify product ID after creation |
| `product_url` | TEXT | URL to product in Shopify |
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
| `published` | Successfully created in Shopify |
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
| `shopify_product_id` | BIGINT | Shopify product ID (unique) |
| `stock_code` | VARCHAR(100) | Leader stock code |
| `sku` | VARCHAR(100) | Shopify SKU |
| `original_raw_title` | VARCHAR(500) | Raw title from Leader |
| `published_title` | VARCHAR(500) | Cleaned title we published |
| `published_slug` | VARCHAR(500) | URL slug from Shopify |
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
- Product must NOT exist in Shopify (`tl_shopify_products_mirror`)
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

**Purpose:** Identifies Leader categories without a target-category mapping.

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
tl_shopify_products_mirror (existing) <────────┘
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

### 5. tl_attribute_mapping

The live **attribute dictionary** — the WC→Shopify crosswalk that replaced the dropped `tl_wc_attributes_mirror` and `tl_wc_attribute_terms_mirror` mirrors after the Phase 13 WooCommerce decommission. `wc_attribute_id` / `wc_slug` are retained as stable **lookup keys** (e.g. `pa_colour`); they no longer reference any WooCommerce table. Owned by the TechLoop Product Enrichment Engine project.

| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT PK | |
| `wc_attribute_id` | INTEGER | Legacy lookup key (nullable); stable join key for `tl_product_attributes` / `tl_proposed_attributes` |
| `wc_slug` | TEXT | Attribute slug, used as the display label (e.g. `pa_colour`) — there is no separate display-name column |
| `shopify_namespace` | TEXT | Metafield namespace (`shopify` / `custom`) |
| `shopify_key` | TEXT | Shopify metafield key |
| `shopify_type` | TEXT | Shopify metafield type (e.g. `single_line_text_field`, `list.metaobject_reference`) |
| `is_promoted` | BOOLEAN | |
| `fan_out` | JSONB | Optional value fan-out rules |
| `normalize_rules` | JSONB | Optional value normalisation rules |
| `status` | TEXT | `active` / disabled |
| `notes` | TEXT | |
| `created_at` / `updated_at` | TIMESTAMPTZ | |

---

### 6. tl_description_models

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

### 7. tl_proposed_attributes

LLM-proposed new attributes or terms awaiting HITL approval via Slack.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `proposal_type` | VARCHAR(50) | `new_attribute` / `new_term` |
| `attribute_name` | VARCHAR(255) | For new attributes |
| `wc_attribute_id` | INTEGER | Legacy lookup key into `tl_attribute_mapping` (FK to the dropped WC mirror removed) |
| `term_value` | VARCHAR(255) | For new terms |
| `product_count` | INTEGER | How many products need this |
| `example_products` | JSONB | Sample stock_codes |
| `status` | VARCHAR(50) | `proposed` / `approved` / `rejected` / `mapped` |


---

### 8. tl_product_attributes

Per-product attribute extraction results. Maps products to attribute keys (via `tl_attribute_mapping`) with confidence scores.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `shopify_product_id` | BIGINT | Shopify product ID |
| `stock_code` | VARCHAR(100) | Leader stock code |
| `wc_attribute_id` | INTEGER | Lookup key into `tl_attribute_mapping` (legacy; FK to the dropped WC mirror removed) |
| `shopify_key` | TEXT | Shopify-native metafield key (alternative identity to `wc_attribute_id`) |
| `raw_value` | TEXT | As extracted by LLM |
| `normalized_value` | TEXT | Matched to existing term/value |
| `confidence` | NUMERIC(3,2) | 0.00–1.00 |
| `source` | VARCHAR(50) | `llm_extraction` / `leader_feed` / `manual` |
| `extraction_run` | INTEGER | Increments on re-extraction |
| `review_status` | VARCHAR(50) | `pending` / `approved` / `rejected` / `pushed_to_shopify` |
| `pushed_to_shopify_at` | TIMESTAMPTZ | When approved values were pushed to Shopify |
| `sent_to_slack_at` | TIMESTAMPTZ | When notification was sent to `#enrichment-review` *(Added 2026-02-21)* |
| `slack_notification_count` | INTEGER | Number of times sent to Slack (default 0) *(Added 2026-02-21)* |

*(The live table also carries Shopify-native enrichment columns — `shopify_metaobject_gid`, `vocab_resolved` — added by the Product Enrichment Engine; see the live schema for the full set.)*

**Partial UNIQUE indexes** *(Phase 13 — `wc_product_id`→`shopify_product_id`; allows multiple values per attribute per product)*:
- `idx_product_attrs_attr_unique` — `(shopify_product_id, wc_attribute_id, normalized_value)` WHERE `wc_attribute_id IS NOT NULL`
- `idx_product_attrs_shopify_native_unique` — `(shopify_product_id, shopify_key, normalized_value)` WHERE `shopify_key IS NOT NULL`

**Notification Tracking (Added 2026-02-21):**
- `sent_to_slack_at` tracks when the item was last sent to Slack for review
- `slack_notification_count` increments each time a notification is sent
- `TL_Enrichment_Reviewer` only sends items where `sent_to_slack_at IS NULL` OR `sent_to_slack_at < NOW() - INTERVAL '1 day'`
- This prevents duplicate Slack messages every 4 hours for the same pending item

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
tl_attribute_mapping  (attribute dictionary: wc_attribute_id / wc_slug → shopify_namespace/key/type)
        |
        ├── lookup key (wc_attribute_id)
        |   v
        |  tl_product_attributes ←── TL_Enrich_Attributes (LLM extraction)
        |       partial-unique (shopify_product_id, wc_attribute_id, normalized_value) WHERE wc_attribute_id IS NOT NULL
        |       partial-unique (shopify_product_id, shopify_key, normalized_value)     WHERE shopify_key IS NOT NULL
        |       review_status: pending → approved → pushed (pushed_to_shopify_at)
        |
        └── lookup key (wc_attribute_id)
            v
           tl_proposed_attributes  (HITL proposals; status: proposed → approved/rejected/mapped)

tl_manufacturer_raw_data ←── TL_Scraper (Crawl4AI — Playwright, stealth mode)
        |  UNIQUE(stock_code, source_url) — multiple rows per product supported
        |  markdown_content → feeds TL_Enrich_Attributes + TL_Enrich_Descriptions
        |  scraped_images   → feeds TL_Enrich_Images
        |  raw_html         → archival (scrape once, parse many times)

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
