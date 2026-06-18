# Workflow 6: Product Publisher (TL_Product_Publisher)

**Status:** Production
**n8n Name:** `TL_Product_Publisher`
**n8n ID:** `xBSqJjECViFMgUNI`
**Direct Link:** [Open in n8n](https://workflows.labgregor.dev/workflow/xBSqJjECViFMgUNI)
**Schedule:** Every 5 minutes
**Cron:** `*/5 * * * *`
**Last Updated:** 2026-02-07

---

## Purpose

The Product Publisher takes **approved products** from the onboarding queue and creates them in WooCommerce. It performs three key processing steps before publishing:

1. **AI Title Optimisation** — Uses Qdrant RAG + Ollama (`qwen2.5:14b`) to generate SEO-friendly titles with rule-based fallback
2. **AI Description Cleanup** — Uses Ollama (`qwen2.5:14b`) to reformat raw supplier specs into clean HTML product descriptions with `<h3>` section headings
3. **AI Short Description** — Uses Qdrant RAG + Ollama (`qwen2.5:14b`) to generate bullet-point short descriptions
4. **Image Processing** — Downloads the product image, adds white padding to make it square, and uploads to WooCommerce Media Library

The critical requirement is that product titles must be **cleaned and optimised** before publishing, as WordPress generates the URL slug from the initial product name.

---

## Prerequisites

- [[Workflow 5 - Product Detector]] must have populated `tl_onboarding_queue`
- Products must have `status = 'approved'`
- Category mappings must exist in `tl_category_map`
- WooCommerce REST API credentials configured in n8n (`httpBasicAuth`)
- Ollama API credential configured in n8n (`ollamaApi`, URL: `http://192.168.1.111:11434`)
- Qdrant API credential configured in n8n (`qdrantApi`, URL: `http://192.168.1.111:6333`)
- Qdrant `product_titles` collection populated (see [[#Qdrant Setup]])
- Qdrant `product_short_descriptions` collection populated (see [[#Qdrant Setup]])
- Ollama models available: `qwen2.5:14b` (title/description generation), `nomic-embed-text` (embeddings)

---

## Critical Requirement: Title Before Slug

**WordPress generates the URL slug from the product name at creation time.**

This means:
1. We must clean the title BEFORE calling the WooCommerce API
2. A raw title like `ASUS ROG STRIX G16 16" 240HZ FHD+ I7-13650HX RTX 4060 16GB 1TB SSD W11H 2YR` would create an ugly URL
3. We want: `asus-rog-strix-g16-gaming-laptop-16-i7-13650hx-rtx-4060`

---

## Workflow Logic

### Product Selection

1. Fetch approved products ordered by priority (highest first)
2. Lock the product (`processing_started_at = NOW()`)
3. Re-fetch fresh data from `tl_feed_leader_raw` to get current stock/price
4. Process title optimisation (AI with rule-based fallback)
5. Process description cleanup (AI with raw fallback)
6. Generate short description (AI with Qdrant RAG examples)
7. Download and process product image (square with white padding)
8. Create product in WooCommerce
9. Update queue with result
10. Upsert published title back to Qdrant

### Title Optimisation (RAG Approach)

1. Generate embedding of raw title using Ollama `nomic-embed-text`
2. Query Qdrant vector database for 5 similar existing product titles
3. Provide examples to Ollama `qwen2.5:14b` with optimisation rules
4. AI generates a clean, SEO-optimised title (max 80 characters)
5. Fallback: Simple rule-based cleaning if AI fails or returns invalid output

### Description Cleanup (AI Approach)

1. Send raw `product_name_2` from Leader to Ollama `qwen2.5:14b`
2. AI reformats into clean HTML with grouped spec lists
3. Uses `<h3>` for section headings (not `<h2>`, as the description renders inside an existing `<h2>`)
4. Only outputs sections where relevant data exists (no empty/fabricated sections)
5. For minimal source data (e.g., just a product name), outputs a brief paragraph instead of padded sections
6. Fallback: Use raw `product_name_2` as-is if AI fails

### Short Description (AI + RAG Approach)

1. Generate embedding of optimised title using Ollama `nomic-embed-text`
2. Query Qdrant `product_short_descriptions` collection for 3 similar existing short descriptions
3. Provide examples to Ollama `qwen2.5:14b` with short description rules
4. AI generates an HTML bullet list (`<ul><li>`) with 4-7 key specs
5. Fallback: Empty short description if AI fails

### Image Processing

1. Download product image from Leader `image_url`
2. Get image dimensions via Edit Image (information)
3. Calculate canvas size: `longest_side * 1.2` (10% padding each side), capped at 2048
4. Add white border to create square image (Edit Image)
5. Resize if over 2048x2048 (Edit Image)
6. Upload processed image to WooCommerce Media Library (`/wp-json/wp/v2/media`)
7. Use returned media attachment ID in product body (instead of URL sideloading)

---

## Node Architecture

| # | Node Type | Node Name | Purpose |
|---|-----------|-----------|---------|
| 1 | Schedule Trigger | Schedule Every 30 Minutes | Cron: `*/30 * * * *` |
| 2 | Code | Capture Start Time | Record workflow start for duration tracking |
| 3 | Postgres | Fetch Approved Products | Get approved products with stale lock timeout |
| 4 | IF | Has Approved? | Check if any approved products found |
| 5 | Postgres | Lock Product | Set `status = 'processing'` |
| 6 | Postgres | Get Fresh Leader Data | Re-fetch from `tl_feed_leader_raw` |
| 7 | IF | Still In Stock? | Verify product still available |
| 8 | **Qdrant Vector Store** | Search Similar Titles | Built-in Qdrant node (load mode) — searches for 5 similar titles |
| 8a | **Embeddings Ollama** | Embeddings Ollama | Sub-node (`nomic-embed-text`) — generates embeddings for Qdrant search |
| 8b | Code | Format Title Prompt | Aggregates Qdrant results into prompt for Ollama |
| 9 | **Ollama** | AI Optimise Title | Built-in Ollama node (`qwen2.5:14b`) with system prompt |
| 10 | Code | Parse AI Title | Extract title, validate, fallback to rule-based |
| 11 | **Ollama** | AI Clean Description | Built-in Ollama node (`qwen2.5:14b`) for HTML description |
| 12 | Code | Parse Clean Description | Extract HTML, validate, fallback to raw |
| 12a | **Qdrant Vector Store** | Search Similar Short Descs | Built-in Qdrant node (load mode) — searches for 3 similar short descriptions |
| 12b | **Embeddings Ollama** | Embeddings Ollama SD | Sub-node (`nomic-embed-text`) — generates embeddings for short desc search |
| 12c | Code | Format Short Desc Prompt | Aggregates Qdrant results into prompt for Ollama |
| 12d | **Ollama** | AI Short Description | Built-in Ollama node (`qwen2.5:14b`) for bullet-point short description |
| 12e | Code | Parse Short Description | Extract HTML, validate `<ul>` structure |
| 13 | Code | Calculate Price | RRP as regular price, DBP inc GST as cost |
| 14 | Postgres | Lookup Brand | Query `tl_brand_map` for WC brand term ID |
| 15 | Code | Merge Brand Data | Merge brand from queue/map/raw |
| 16 | IF | Has Brand ID? | Route to create brand if missing |
| 17 | HTTP Request | Create Brand If Missing | POST to WC brands API |
| 18 | Code | Resolve Brand ID | Capture brand ID from creation |
| 19 | HTTP Request | Download Image | GET image from Leader URL (binary) |
| 20 | IF | Has Image? | Check if image download succeeded |
| 21 | Edit Image | Get Image Info | Read image dimensions |
| 22 | Code | Calculate Padding | Compute border sizes for square + 10% padding |
| 23 | Edit Image | Add White Border | Add calculated white border |
| 24 | Edit Image | Cap Size | Resize if over 2048x2048 |
| 25 | HTTP Request | Upload to WC Media | POST to `/wp-json/wp/v2/media` |
| 26 | Code | Merge Image Result | Collect media ID or fallback to URL sideloading |
| 27 | Code | Build WC Product Body | Construct full WooCommerce product JSON |
| 28 | HTTP Request | Create WooCommerce Product | POST to WC Products API |
| 29 | IF | Handle Result | Route success/failure |
| 30 | Postgres | Mark Published | Update queue: `status = 'published'` |
| 31 | Postgres | Create Optimization Record | INSERT into `tl_product_optimizations` |
| 32a | Code | Prepare Title Document | Format published title as document for Qdrant |
| 32b | **Qdrant Vector Store** | Upsert Title to Qdrant | Built-in Qdrant node (insert mode) — upserts title to collection |
| 32c | **Embeddings Ollama** | Embeddings Ollama 1 | Sub-node (`nomic-embed-text`) — generates embeddings for upsert |
| 33 | Postgres | Mark Failed | Update queue: `status = 'failed'` |
| 34 | Postgres | Mark Out of Stock | Mark product out of stock |
| 35 | Code | Build Summary | Aggregate results for email |
| 36 | Postgres | Log Execution | Log to `tl_workflow_executions` |
| 37 | IF | Published Any? | Check if email needed |
| 38 | Code | Prepare Email | Build styled HTML email |
| 39 | SMTP | Send Email | Send via Elastic Email |
| 40 | NoOp | No Products | Terminal: no approved products |
| 41 | NoOp | No Email | Terminal: nothing published |

---

## Node Details

### Node 1: Schedule Every 30 Minutes

```
Trigger: Cron
Expression: */30 * * * *
```

Runs every 30 minutes to process approved products.

---

### Node 3: Fetch Approved Products (Postgres)

**Operation:** Execute Query

```sql
SELECT * FROM tl_onboarding_queue
WHERE status = 'approved'
  AND (processing_started_at IS NULL
       OR processing_started_at < NOW() - INTERVAL '10 minutes')
ORDER BY priority DESC, detected_at ASC
LIMIT 1;
```

**Note:** Stale locks (>10 minutes) are reclaimed automatically.

---

### Node 8: Search Similar Titles (Built-in Qdrant Vector Store)

**Node Type:** `@n8n/n8n-nodes-langchain.vectorStoreQdrant` (load mode, `main` connections)
**Credential:** `qdrantApi` (URL: `http://192.168.1.111:6333`)
**Collection:** `product_titles`
**Prompt:** `{{ $json.product_name }}` (raw title from Leader data)
**Limit:** 5 results
**Error Handling:** `onError: continueRegularOutput` (graceful fallback)

**Sub-node: Embeddings Ollama** (`nomic-embed-text`) generates the embedding vector for the search query.

**Followed by: Format Title Prompt** (Code) — aggregates the Qdrant results into a prompt string with examples for the AI title optimiser.

---

### Node 9: AI Optimise Title (Built-in Ollama Node)

**Node Type:** `@n8n/n8n-nodes-langchain.ollama` (standalone, `main` connections)
**Model:** `llama3.2:latest`
**Credential:** `ollamaApi` (URL: `http://192.168.1.111:11434`)

**System prompt:**
```
You are a product title optimiser for TechLoop, an Australian tech retailer.

Rules:
- Keep brand name first
- Include key model name/number
- Include key specs (CPU, GPU, screen size for laptops; capacity for storage; etc.)
- Maximum 80 characters
- Remove warranty info, Windows version, redundant specs, weight
- Use proper capitalisation
- Australian English spelling
- Do NOT wrap in quotes
```

**User message:** `{{ $json.title_prompt }}` (built by Search Similar Titles)

**Settings:** temperature=0.3, num_predict=100

---

### Node 10: Parse AI Title (Code)

- Handles multiple Ollama output formats (`text`, `message.content`, `response`)
- Validates AI output (10-100 characters)
- Falls back to rule-based cleaning if AI fails
- Records `title_source: 'ai'` or `'rule_based'`

---

### Node 11: AI Clean Description (Built-in Ollama Node)

**Node Type:** `@n8n/n8n-nodes-langchain.ollama` (standalone, `main` connections)
**Model:** `llama3.2:latest`
**Credential:** `ollamaApi`

**System prompt:**
```
You are a product description writer for TechLoop, an Australian tech retailer.

Reformat this raw product specification text into a clean, readable product description.

Rules:
- Use HTML formatting (paragraphs, bullet lists)
- Group specs into logical sections where appropriate (e.g., Display, Performance, Storage, Connectivity — but only use sections that are relevant to the product)
- Do NOT output a section if there is no relevant information for it in the source data
- Do NOT invent or fabricate information that is not in the original text
- Use <h3> for section headings (NOT <h1> or <h2> — the description is displayed inside an existing <h2> element)
- Use <ul><li> for spec lists
- Keep it factual, no marketing fluff
- Australian English spelling
- Keep it concise
- If the source data is minimal (e.g., just a product name), output a brief paragraph — do not pad with empty or irrelevant sections
```

**User message:** Raw description + product name + brand (from previous nodes)

**Settings:** temperature=0.3, num_predict=2048

---

### Node 12: Parse Clean Description (Code)

- Handles multiple Ollama output formats
- Strips markdown code block wrappers
- Validates output contains HTML tags and minimum length
- Falls back to raw `product_name_2` if AI fails

---

### Nodes 20-27: Image Processing Chain

**Node 20 — Download Image:** GET request to Leader `image_url`, response as binary file.

**Node 21 — Has Image?:** Checks if binary data exists from download.

**Node 22 — Get Image Info:** Edit Image node (operation: `information`) reads width and height.

**Node 23 — Calculate Padding:**
```javascript
const longest = Math.max(width, height);
let canvas = Math.round(longest * 1.2);  // 10% padding each side
if (canvas > 2048) canvas = 2048;
const hBorder = Math.round((canvas - width) / 2);
const vBorder = Math.round((canvas - height) / 2);
```

**Node 24 — Add White Border:** Edit Image node (operation: `border`) with white `#FFFFFF` background.

**Node 25 — Cap Size:** Edit Image node (operation: `resize`, `onlyIfLarger`) to max 2048x2048.

**Node 26 — Upload to WC Media:**
```
POST https://techloop.com.au/wp-json/wp/v2/media
Headers: Content-Disposition: attachment; filename="{{ stock_code }}.png"
Body: Binary image data
Auth: WooCommerce API (Basic Auth)
```

**Node 27 — Merge Image Result:** Collects `wc_media_id` from upload response, or falls back to URL sideloading.

---

### Node 28: Build WC Product Body

Constructs the full WooCommerce product JSON including:
- AI-optimised title as `name`
- AI-cleaned HTML description
- Uploaded media attachment ID (or fallback to image URL sideloading)
- RRP as `regular_price`, DBP inc GST as COGS
- Category, brand, attributes (Brand, Model Number, Warranty)
- ACF custom fields with field reference keys
- Status: `draft`

---

### Node 32: Upsert Title to Qdrant (Built-in Qdrant Vector Store)

After successful publishing, the new title is prepared as a document and upserted to Qdrant:

1. **Prepare Title Document** (Code) — formats the published title as `pageContent` with metadata (brand, SKU, WC product ID)
2. **Upsert Title to Qdrant** — Built-in Qdrant Vector Store node (insert mode), `onError: continueRegularOutput`
3. **Embeddings Ollama 1** — Sub-node (`nomic-embed-text`) generates embedding for the upsert

---

## Qdrant Setup

### Seed Workflow: TL_Seed_Qdrant_Titles

**n8n ID:** `WSZvDoxgUBmOuM8W`
**Purpose:** One-time utility to populate Qdrant with existing good product titles from `tl_wc_products_mirror`.

**Flow:** Manual Trigger → Fetch Good Titles (Postgres) → Prepare Documents (Code) → Insert Titles to Qdrant (Built-in Qdrant Vector Store, insert mode) + Embeddings Ollama sub-node (`nomic-embed-text`)

The `collectionConfig` option auto-creates the `product_titles` collection (768 dimensions, Cosine distance) if it doesn't exist.

**SQL for good titles:**
```sql
SELECT wc_id AS wc_product_id, name AS title, brand_name, sku
FROM tl_wc_products_mirror
WHERE name IS NOT NULL AND status = 'publish'
ORDER BY wc_id;
```

### Seed Workflow: TL_Seed_Qdrant_Short_Descriptions

**n8n ID:** `2M95azR2qs49Tku4`
**Purpose:** One-time utility to populate Qdrant with existing product short descriptions from `tl_wc_products_mirror`.

The `collectionConfig` option auto-creates the `product_short_descriptions` collection (768 dimensions, Cosine distance) if it doesn't exist.

### Qdrant Collection: product_titles

```bash
curl -X PUT http://192.168.1.111:6333/collections/product_titles \
  -H "Content-Type: application/json" \
  -d '{
    "vectors": { "size": 768, "distance": "Cosine" },
    "payload_schema": {
      "title": { "data_type": "Text" },
      "brand": { "data_type": "Text" }
    }
  }'
```

---

## WooCommerce Field Mapping Reference

| WooCommerce Field | Source | Notes |
|-------------------|--------|-------|
| `name` | AI-optimised title | From Ollama + Qdrant RAG |
| `slug` | Auto-generated | WooCommerce creates from name |
| `description` | AI-cleaned HTML | From Ollama, fallback to raw. Uses `<h3>` headings |
| `short_description` | AI-generated bullet list | From Ollama + Qdrant RAG, fallback to empty |
| `sku` | `manufacturer_sku` or `stock_code` | Leader identifiers |
| `regular_price` | `rrp` | Rounded to nearest dollar |
| `stock_quantity` | `availability_total` | Re-fetched at publish |
| `stock_status` | `instock` / `outofstock` | Based on availability |
| `manage_stock` | `true` | Always manage stock |
| `status` | `draft` | Review before making live |
| `categories` | `[{ id: mapped_wc_category_id }]` | From category map |
| `images` | `[{ id: wc_media_id }]` | Uploaded processed image (square, padded) |
| `weight` | `weight` | Parsed if numeric |
| `dimensions` | `length/width/height` | Converted mm to cm |
| `brands` | `[{ id: brand_term_id }]` | Brand taxonomy |
| `cost_of_goods_sold` | `dbp_inc_gst` | For COGS tracking |
| `meta_data` | ACF fields | See ACF mapping below |

### ACF Fields Mapping

| ACF Key | Source | Purpose |
|---------|--------|---------|
| `supplier_name` | `"Leader"` | Supplier identifier |
| `supplier_product_id` | `stock_code` | Links to Leader feed |
| `supplier_dbp_ex_gst` | `dbp_ex_gst` | Cost (Ex GST) |
| `supplier_dbp_inc_gst` | `dbp_inc_gst` | Cost (Inc GST) |
| `supplier_mpn` | `manufacturer_sku` | Manufacturer Part Number |
| `supplier_barcode` | `bar_code` | EAN/GTIN |
| `_wc_gtin` | `bar_code` | Google Shopping GTIN |

---

## Error Handling

### Retry Logic

Products that fail are marked with `status = 'failed'` and `retry_count` incremented.

Failed products with `retry_count < 3` will be retried on subsequent runs.

### AI Fallbacks

| Feature | Primary | Fallback |
|---------|---------|----------|
| Title | Ollama + Qdrant RAG | Rule-based regex cleaning |
| Description | Ollama HTML generation | Raw `product_name_2` as-is |
| Short Description | Ollama + Qdrant RAG | Empty string |
| Image | Download → Process → Upload | URL sideloading (`src: image_url`) |

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `401 Unauthorized` | WooCommerce API credentials | Check n8n credentials |
| `500 Internal Server Error` | WooCommerce issue | Check WC error logs |
| `duplicate_sku` | Product already exists | Check `tl_wc_products_mirror` |
| `Image download failed` | Leader image URL invalid | Continue without image |
| `Category not found` | Invalid WC category ID | Update `tl_category_map` |
| `Ollama timeout` | Model not loaded or slow | Title/desc falls back automatically |
| `Qdrant connection refused` | Qdrant not running | Title falls back to rule-based |

---

## Testing

### Manual Test

1. Approve a product in Supabase:
   ```sql
   UPDATE tl_onboarding_queue
   SET status = 'approved'
   WHERE stock_code = 'TEST123'
   LIMIT 1;
   ```

2. Manually trigger workflow in n8n

3. Verify in WooCommerce:
   - Product created as draft
   - Title is clean (check `title_source` in optimizations table)
   - Description is formatted HTML
   - Image is square with white padding
   - URL slug is readable
   - All ACF fields populated
   - COGS filled

### Verification Queries

```sql
-- Check published products
SELECT stock_code, product_name, wc_product_id, wc_product_url, published_at
FROM tl_onboarding_queue
WHERE status = 'published'
ORDER BY published_at DESC
LIMIT 10;

-- Check optimizations record created
SELECT wc_product_id, original_raw_title, published_title, published_slug
FROM tl_product_optimizations
ORDER BY created_at DESC
LIMIT 10;

-- Check for failed publishes
SELECT stock_code, product_name, retry_count, publish_error
FROM tl_onboarding_queue
WHERE status = 'failed'
ORDER BY updated_at DESC;

-- Check Qdrant has titles
-- curl http://192.168.1.111:6333/collections/product_titles
```

---

## Related Documentation

- [[Workflow 5 - Product Detector]]
- [[Category Mapping Guide]]
- [[Database Schema - Product Automation]]
- [[System Overview - TechLoop Inventory Automation]]
