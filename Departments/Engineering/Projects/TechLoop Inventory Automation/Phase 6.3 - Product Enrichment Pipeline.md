# Phase 6.3: Product Enrichment Pipeline

**Status:** 🚧 In Progress
**Prerequisite:** Phase 6.1-6.2 ✅ Complete
**Last Updated:** 2026-02-22

### Phase 0 — Scraping Pipeline ✅ Complete (2026-02-22)

- Browserless Chromium Docker container deployed at `192.168.1.111:3100` (compose file: `~/browserless/docker-compose.yml`)
- Schema migrated: `tl_manufacturer_raw_data` table + 6 new columns on `tl_product_optimizations`
- `TL_URL_Discoverer` workflow live (ID: `QQMwVzR5c79Nx3RL`) — daily at 02:00 AEST
- `TL_Scraper` workflow live (ID: `b4rR1gwQ25uUJkEQ`) — every 2 hours, 5 products/run
- 791 products seeded with `scrape_status = 'pending'` from Leader feed URLs
- Upserts use Supabase REST API (anon key) for safe handling of large HTML payloads

---

## Overview

Phase 6.3 extends the product automation system with a full enrichment pipeline: manufacturer data scraping, structured attribute extraction, feature-focused descriptions, and high-quality image sourcing. It builds on the existing `tl_product_optimizations` table with its three independent status tracks (`description_status`, `image_status`, `attribute_status`).

### Core Principles

- **Decoupled:** Scraping → staging → AI processing → HITL review → WC push
- **Database-as-queue:** All n8n workflows read/write Supabase directly
- **WooCommerce is source of truth for attributes** — Supabase mirrors it
- **Re-runnable:** New data sources automatically trigger re-extraction
- **Specs are sacred:** AI curates and structures, never fabricates (see [[Tone of Voice]] Rule 3 and [[AI Writing Guidelines]])

### LLM Strategy

- **Attributes:** Ollama `qwen2.5:14b` (local, free, proven)
- **Descriptions:** Configurable per run via `tl_description_models` table — supports Ollama, OpenRouter, Anthropic Claude

---

## Architecture

```
                    WooCommerce (Source of Truth for Attributes)
                     ├── /products/attributes         (global definitions)
                     ├── /products/attributes/<id>/terms  (values/options)
                     └── /products/<id>  { attributes }  (per-product)
                                    ▲ push            │ mirror
                                    │                 ▼
              ┌──────────────────────────────────────────────────────┐
              │                 Supabase (Local State)                │
              │  tl_wc_attributes_mirror    ← attribute definitions  │
              │  tl_wc_attribute_terms_mirror ← terms per attribute  │
              │  tl_wc_products_mirror.attributes ← product attrs    │
              │  tl_product_attributes       → proposed/approved      │
              └──────────────────────────────────────────────────────┘
                                    ▲
Phase 0: Scraping         Phase 1: Attributes      Phase 2: Descriptions
┌──────────────────┐     ┌─────────────────────┐   ┌──────────────────┐
│ TL_Scraper       │────▶│ TL_Enrich_Attributes│──▶│ TL_Enrich_       │
│ TL_URL_Discoverer│     │ TL_Attribute_Proposer│  │   Descriptions   │
└──────────────────┘     │ TL_Attribute_WC_Push │  └──────────────────┘
         │               └─────────────────────┘
         ▼                        │                Phase 3: Images
┌──────────────────┐              ▼                ┌──────────────────┐
│ tl_manufacturer_ │     ┌─────────────────────┐   │ TL_Enrich_Images │
│ raw_data         │     │ HITL (#enrichment-  │   └──────────────────┘
│ (full page HTML) │     │       review)       │
│ scrape once,     │     │ Q1: New attributes  │
│ parse many times │     │ Q2: New terms       │
└──────────────────┘     │ Q3: Product mappings│
                         └─────────────────────┘
```

---

## New Database Tables

### tl_manufacturer_raw_data (Data Lake)

Stores full scraped content from manufacturer pages. Scrape once, parse many times — future workflows (reviews, videos) re-parse the same `raw_html`. Supports multiple rows per product via `UNIQUE(stock_code, source_url)`, ready for multi-page scraping (e.g. main + `/sp` + `/gallery`).

**Scraper:** Crawl4AI (Playwright-based, stealth mode) at `192.168.1.111:11235`. Replaced Browserless 2026-02-28.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `stock_code` | VARCHAR(100) | Links to `tl_feed_leader_raw` |
| `source_url` | TEXT | Manufacturer page URL |
| `url_source` | VARCHAR(50) | `leader_feed` / `serp_discovery` / `manual` |
| `raw_html` | TEXT | Full page HTML (archival — scrape once, parse many times) |
| `markdown_content` | TEXT | Clean Markdown from Crawl4AI — primary source for all LLM enrichment |
| `scraped_images` | JSONB | `[{src, alt, tier, confidence, format}]` extracted by Crawl4AI |
| `scraped_videos` | JSONB | `[{src, type, title}]` |
| `scrape_status` | VARCHAR(50) | `pending` / `completed` / `failed` / `skipped` |
| `scrape_method` | VARCHAR(50) | `crawl4ai` |
| `retry_count` | INTEGER | Default 0 |

**UNIQUE:** `(stock_code, source_url)`

**Dropped 2026-02-28** (redundant with Crawl4AI Markdown): `raw_text`, `spec_tables_html`, `feature_sections_html`

### tl_wc_attributes_mirror

Mirror of WooCommerce global attributes (`GET /products/attributes`).

| Column | Type | Description |
|--------|------|-------------|
| `wc_attribute_id` | INTEGER PK | From WooCommerce |
| `name` | VARCHAR(255) | e.g., "RAM", "Screen Size" |
| `slug` | VARCHAR(255) UNIQUE | |
| `type` | VARCHAR(50) | `select` |
| `category_scope` | TEXT[] | Which WC categories this applies to (local metadata) |
| `unit` | VARCHAR(50) | e.g., "GB", "GHz", "inch" (local metadata) |
| `last_synced_at` | TIMESTAMPTZ | |

### tl_wc_attribute_terms_mirror

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

### tl_proposed_attributes

LLM-proposed new attributes or terms awaiting HITL approval.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | |
| `proposal_type` | VARCHAR(50) | `new_attribute` / `new_term` |
| `attribute_name` | VARCHAR(255) | For new attributes |
| `wc_attribute_id` | INTEGER | For new terms (which attribute) |
| `term_value` | VARCHAR(255) | For new terms |
| `product_count` | INTEGER | How many products need this |
| `example_products` | JSONB | Sample stock_codes |
| `status` | VARCHAR(50) | `proposed` / `approved` / `rejected` / `mapped` |

### tl_attribute_aliases

Maps alternative attribute names to canonical WC attributes. Prevents the LLM from proposing duplicates when manufacturer specs use different names for the same attribute (e.g., "Memory" vs "RAM", "M/B Type" vs "Motherboard Form Factor").

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID PK | Default `gen_random_uuid()` |
| `alias_name` | VARCHAR(255) UNIQUE | The alternative name |
| `maps_to_wc_attribute_id` | INTEGER NOT NULL FK | → `tl_wc_attributes_mirror` |
| `source` | VARCHAR(50) | `hitl_mapping` / `hitl_rejection` / `manual` |
| `created_at` | TIMESTAMPTZ | Default `NOW()` |

**Created via:** HITL "Map to Existing" dropdown in `TL_Attribute_Proposer` Slack messages, or manual DB insert.

**Consumed by:** `TL_Enrich_Attributes` — aliases are LEFT JOINed into the attribute dictionary and included in the LLM prompt. The parser also resolves aliases at runtime as a fallback.

### tl_product_attributes

Per-product attribute extraction results.

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
| `slack_notification_count` | INTEGER | Times sent to Slack, default 0 *(Added 2026-02-21)* |

**UNIQUE:** `(wc_product_id, wc_attribute_id, normalized_value)` *(Updated 2026-02-22 — supports multiple values per attribute per product)*

### tl_description_models

Configurable LLM models for description generation.

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

### Modifications to Existing Tables

**`tl_product_optimizations`** — new columns:
- `manufacturer_url` TEXT
- `manufacturer_url_source` VARCHAR(50)
- `scrape_status` VARCHAR(50) DEFAULT `pending`
- `description_version` INTEGER DEFAULT 1
- `description_model_id` UUID FK → `tl_description_models`
- `image_error` TEXT

**`tl_wc_products_mirror`** — new column:
- `attributes` JSONB — mirrors WC product attributes array

---

## Workflow Catalogue

| Workflow | Concern | Schedule |
|----------|---------|----------|
| `TL_Mirror_WC_Attributes` | Mirror WC attribute definitions + terms | Daily |
| `TL_Enrich_Attributes` | LLM extraction → `tl_product_attributes` | Every 15 min |
| `TL_Attribute_Proposer` | Group unknown attrs/terms → HITL Slack | Daily |
| `TL_Attribute_WC_Pusher` | Push approved mappings to WC products | Every 10 min |
| `TL_Scraper` | Fetch manufacturer pages → data lake | Every 2 hours |
| `TL_URL_Discoverer` | Find manufacturer URLs | Daily |
| `TL_Enrich_Descriptions` | Feature-focused descriptions | Every 15 min |
| `TL_Enrich_Images` | Process scraped images → WC gallery | Every 30 min |
| `TL_Enrichment_Reviewer` | Batch HITL review via Slack | Every 4h + slash |

**Docker service:** Browserless (Chromium) on `192.168.1.111:3100` — JS rendering for heavy sites.

**Existing workflows modified:**
- `TL_Mirror_WooCommerce` — add `attributes` JSONB extraction
- `TL_Slack_Interaction_Handler` — add enrichment action IDs

---

## Attribute System (4 Layers)

### Layer 1: WC Mirror
`TL_Mirror_WC_Attributes` syncs WC's global attributes and terms to Supabase daily. Independent of all other workflows.

### Layer 2: LLM Extraction
`TL_Enrich_Attributes` uses Ollama to extract structured attributes from product data. Maps to existing WC attributes/terms where possible. Flags unknowns for HITL.

**Data sources** (in priority order):
1. `spec_tables_html` from manufacturer scrape (richest)
2. `product_name_2` from Leader feed (always available)
3. Product title and category (supplementary)

**Alias resolution:** The attribute dictionary query LEFT JOINs `tl_attribute_aliases` so aliases appear in the LLM prompt as "(also known as: Memory, System Memory)". The parser also builds a runtime alias lookup — if the LLM returns an alias name, it resolves to the correct WC attribute instead of proposing a new one.

**Multi-value handling:** The prompt instructs the LLM to return one JSON object per value (e.g., separate objects for "ATX" and "microATX" instead of "ATX, microATX"). The parser also splits any remaining comma-separated values as a fallback.

**Re-runnable:** When the scraper writes new data, `attribute_status` resets to `pending` → extraction re-runs with richer context. `extraction_run` increments.

### Layer 3: HITL
`TL_Attribute_Proposer` groups proposed attributes/terms across products and sends batch review to `#enrichment-review`:
- **Queue 1:** New attributes ("500 products have 'Bluetooth Version' — create in WC?")
  - Approve → creates attribute in WC
  - Reject → marks as rejected
  - **Map to Existing** → dropdown of all WC attributes; inserts alias into `tl_attribute_aliases`, marks proposal as `mapped`. Future extractions will use the alias automatically.
- **Queue 2:** New terms ("RAM = '24GB' doesn't exist yet — add to RAM attribute?")

### Layer 4: WC Push
`TL_Attribute_WC_Pusher` pushes approved mappings to WC via `PUT /products/<id>`, merging with existing attributes.

**Auto-approval:** Confidence ≥ 0.90 + existing WC attribute + existing WC term → auto-approve.

### Attribute Naming

Shared attributes across categories get context in their names:
- "Drive Form Factor" (M.2, 2.5", 3.5") vs "Case Form Factor" (ATX, mATX, SFF)
- "Storage Interface" (SATA, NVMe) vs "Network Interface" (RJ45, SFP+)

---

## Description Generation

### Key Principle

Descriptions focus on **features, benefits, and use cases** — NOT technical specs. Specs are handled by the Attribute system and display in WC's "Additional Information" tab.

### Template

```html
<p>{Brief intro — what the product is and who it's for}</p>

<ul>
  <li>{Feature 1 summary}</li>
  <li>{Feature 2 summary}</li>
  <li>{Feature 3 summary}</li>
</ul>

<h4>{Feature Heading 1}</h4>
<p>{Feature description — drawn from manufacturer copy}</p>

<h4>{Feature Heading 2}</h4>
<p>{Feature description}</p>
<!-- 3-6 feature blocks max -->
```

### Quality Standards

- **Voice:** [[Tone of Voice]] — intro gets TechLoop voice, feature blocks preserve manufacturer language
- **Writing:** [[AI Writing Guidelines]] — no AI slop, sound human, cut filler
- **Specs Are Sacred:** Feature claims use manufacturer's exact language. Do not paraphrase technical specs.
- **Model selection:** Configurable via `tl_description_models` table. Supports Ollama, OpenRouter, Anthropic.

---

## Implementation Order

| Week | Deliverable |
|------|-------------|
| 1 | Schema + `TL_Mirror_WC_Attributes` + modify product mirror |
| 2 | `TL_Enrich_Attributes` + `TL_Attribute_Proposer` |
| 3 | `TL_Enrichment_Reviewer` + `TL_Attribute_WC_Pusher` + Slack HITL |
| 4 | `TL_Scraper` + Browserless + re-run triggers |
| 5 | `TL_Enrich_Descriptions` + `TL_Enrich_Images` |

---

## Enrichment Review Improvements (2026-02-21/22)

Three improvements were implemented after the initial pipeline deployment. See [[Implementation - Enrichment Review Improvements]] for full detail.

### 1. Duplicate Slack Message Prevention

`TL_Enrichment_Reviewer` (ID: `OarmHP6DpJvAV0Pb`) was updated to prevent the same pending item being sent to Slack every 4 hours.

The Fetch Pending Mappings query (node-2) was replaced with an atomic CTE that fetches unsent items AND marks them as sent in a single query. Items are only re-sent if `sent_to_slack_at IS NULL` or `sent_to_slack_at < NOW() - INTERVAL '1 day'`.

Two additional fixes were applied:
- IF Has Items (node-3): now checks for a valid `id` field rather than item count, preventing the `{"success": true}` response from an empty CTE result slipping through as a valid item.
- Format Summary (node-9): wrapped in `$if($('Build Review Messages').isExecuted, ...)` to prevent stale data from a previous run being reported when the current run finds no items.
- Slack Summary (node-10): added `onError: "continueRegularOutput"` so intermittent Slack API errors do not fail the whole workflow run.

### 2. Multi-Value Attribute Splitting

`TL_Enrich_Attributes` (ID: `lq8960K0xVwF3Xst`) now correctly handles attributes with multiple values (e.g., a monitor with both DVI-D and HDMI interfaces).

Changes:
- Parse LLM Response (node-8): added `splitMultiValue()` helper with three-pattern detection — "Yes x N (...)" repetition, multiple parenthetical groups, and comma-separated fallback.
- Build LLM Prompts (node-6): explicit instruction added to the CRITICAL section of the prompt telling the LLM to return one JSON object per value.
- Upsert Extractions (node-9): `ON CONFLICT` updated from `(wc_product_id, wc_attribute_id)` to `(wc_product_id, wc_attribute_id, normalized_value)` to match the new database constraint.

### 3. Schema Constraint Update

The unique constraint on `tl_product_attributes` was changed to allow multiple values per attribute per product:
- Old: `UNIQUE(wc_product_id, wc_attribute_id)`
- New: `UNIQUE(wc_product_id, wc_attribute_id, normalized_value)`

---

## Related Documentation

- [[System Overview - TechLoop Inventory Automation]]
- [[Database Schema - Product Automation]]
- [[Future Roadmap - TechLoop Inventory Automation]]
- [[Workflow 6 - Product Publisher]]
- [[Slack Integration]]
- [[Tone of Voice]]
- [[AI Writing Guidelines]]
- [[Ubuntu Server - LabGregor]]

---

## Tags

#project #automation #phase-6 #enrichment #attributes #ai
