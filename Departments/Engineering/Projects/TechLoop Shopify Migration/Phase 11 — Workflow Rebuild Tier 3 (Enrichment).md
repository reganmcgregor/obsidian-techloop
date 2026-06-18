---
status: done
notion_page_id: null
notion_parent_id: 3558c3d3-c03e-8164-9b2c-ff4243c5147a
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# Phase 11 — Workflow Rebuild Tier 3 (Enrichment)

Third tier of the n8n rebuild: push existing Supabase-stored, HITL-reviewed attribute values to Shopify metafields, then rebuild the ongoing enrichment chain against Shopify. See [[TechLoop Shopify Migration]], [[Phase 10 — Workflow Rebuild Tier 2 (Onboarding)]], [[Attribute Strategy (deferred)]].

---

## Core design principles

**Prefer `shopify.*` namespace.** Shopify standard attributes (`shopify.*`) are surfaced in admin, supported by Shopify Magic, and map cleanly to category-level filtering. Wherever a Shopify standard attribute covers a concept, promote to `shopify.*` and deprecate the `custom.*` equivalent. Avoid maintaining both.

**Lazy definition creation.** `shopify.*` metafield definitions are auto-created by Shopify on first `metafieldsSet` write — no DDL step required. Only create explicit `custom.*` definitions for attributes with no Shopify standard equivalent.

**Single push path.** `TL_Attribute_Pusher_Shopify` is used by both the Phase 11.A bulk backfill and the Phase 11.B HITL "approve attribute" button — same workflow, no parallel codebase.

**All per product.** For each product, gather all attribute data in one Supabase query, map to `shopify.*` + `custom.*`, write via a single `metafieldsSet` mutation (accepts an array). No multi-pass or attribute-type batches.

---

## Sub-phases overview

| Sub-phase | Title | Status | Depends on |
|---|---|---|---|
| 11.0 | Attribute Mapping Audit | **Complete** (2026-06-11) | — |
| 11.A | Bulk Backfill | **Complete** (2026-06-17) | 11.0 |
| 11.B | Ongoing Enrichment Rebuild | **Complete** (2026-06-18) | 11.A ✅ |
| 11.C | Targeted Scrape for Gaps | Optional | 11.A audit Sheet |

---

## Phase 11.0 — Attribute Mapping Audit

**What:** A desk-based mapping exercise. For each of the 95 `custom.*` metafield definitions, decide whether it should be promoted to a `shopify.*` standard attribute, kept as `custom.*`, or deprecated.

**Inputs:**
- Supabase: `tl_wc_products_mirror` enrichment columns + all `custom.*` metafield keys in use; top-5 sample values per attribute (script queries Shopify Admin API or Supabase)
- Shopify taxonomy: 3,310 standard product attributes via `taxonomyCategories` query (or static export from `shopify.github.io/product-taxonomy`)

**Process:**
1. Script queries both sources and outputs a Google Sheet with columns: `custom_key` | `sample_values` | `proposed_shopify_handle` | `proposed_shopify_type` | `confidence` | `decision` | `notes`
2. Auto-propose 0–3 candidate `shopify.*` matches per `custom.*` key via fuzzy string match + LLM semantic pass (bias toward promote)
3. Category-level pass: for each Shopify product category in use, compare our attributes against Shopify Magic's category-suggested attributes — auto-propose promote for exact-name matches
4. HITL review in Sheet — mark each row:
   - **promote** → use `shopify.*` namespace; Shopify lazy-creates the def on first write
   - **keep-custom** → no Shopify standard equivalent (e.g. `pcie-slots`, `mtbf`, `fan-airflow-cfm`) — keep `custom.*`
   - **deprecate** → attribute has no values in the live store; drop

**Output:**
- Updated `SHOPIFY_KEY_OVERRIDES` constant (maps WC attribute key → `{namespace, key, type}` in Shopify)
- Deprecation list: `custom.*` defs to remove from Shopify after the backfill (API: `metafieldDefinitionDelete`)
- Net-new `custom.*` defs to create for keep-custom attributes not yet defined in Shopify (minimal — most custom.* defs already exist from Phase 2)

**Effort:** ~3h scripting + ~1h Sheet review + small DDL for keep-custom defs.

**Audit results (2026-06-11):**
- 32 promotes (`shopify.*`) | 138 keep-custom (`custom.*`) | 2 deprecated
- 0 HITL overrides — mapping locked
- Artefacts: `~/Developer/techloop-attribute-audit/output/`
  - `attribute-audit-2026-06-10.csv` — full mapping sheet
  - `SHOPIFY_KEY_OVERRIDES.js` — generated mapping constant
  - `WC_ATTR_ID_TO_SLUG.js` — `wc_attribute_id → slug` lookup
  - `category-gids-cache.json` — 118 TechLoop taxonomy GIDs
- Category Gaps Sheet tab: 283 gaps across 118 categories — awaiting HITL `hitl_action` review
- Google Sheet: `1rbJnCrwZJPeXKjzMSzeFDcS5VshLkG16EDJMsWwGQpY`

---

## Phase 11.A — Bulk Backfill

**What:** Push all existing Supabase attribute values to Shopify metafields across the catalogue. Runs unattended (values were previously HITL-reviewed before landing in WC — no re-review needed).

### Supabase schema (discovered 2026-06-11)

| Table | Key columns | Purpose |
|---|---|---|
| `tl_product_attributes` | `wc_product_id`, `wc_attribute_id`, `normalized_value`, `review_status` | Enrichment pipeline values; push `approved` + `pushed_to_wc` |
| `tl_wc_products_mirror` | `wc_id`, `acf_supplier_product_id`, `attributes` (JSON) | WC product attrs + stock code join key |
| `tl_shopify_products_mirror` | `supplier_product_id`, `shopify_gid` | Shopify GID lookup |
| `tl_wc_attributes_mirror` | `wc_attribute_id`, `slug` | Attr ID → slug (can use static `WC_ATTR_ID_TO_SLUG.js` instead) |

**Join path:** `tl_wc_products_mirror.acf_supplier_product_id` = `tl_shopify_products_mirror.supplier_product_id` → `shopify_gid`

**Data sources (two, merged in Code node):**
1. `tl_wc_products_mirror.attributes` — JSON array of WC product attrs set directly in WC; each has `slug` and `options[]`. All products have this (or null). Lower precedence.
2. `tl_product_attributes WHERE review_status IN ('approved', 'pushed_to_wc')` — HITL-reviewed enrichment values. 1,477 records across 595 products. Higher precedence (overrides WC attrs for same slug).

**~207 WC products not yet in Shopify** (`acf_supplier_product_id` not found in `tl_shopify_products_mirror`) — Pusher returns `skip:not_in_shopify` for these; they'll be processed when onboarded via Phase 10 pipeline.

### TL_Attribute_Pusher_Shopify (n8n workflow)

**Purpose:** Single-product attribute push. Used by both the bulk batch script (11.A) and the ongoing HITL approve button (11.B).

**Input:** `{ wc_product_id: number }` via webhook POST body.

**Reference code:** `~/Developer/techloop-attribute-audit/pusher-logic.js` — contains the full Code node JS, exact Supabase query params, and GraphQL mutation.

**Flow:**
1. **Get Token** — POST `/admin/oauth/access_token`, cred `4Kf8ZBknzZl6bvvi`
2. **Get WC Product** — `tl_wc_products_mirror?wc_id=eq.{id}&select=wc_id,name,acf_supplier_product_id,attributes`
3. **If: No supplier code** — `acf_supplier_product_id` is null → Respond error
4. **Get Shopify GID** — `tl_shopify_products_mirror?supplier_product_id=eq.{code}&select=shopify_id,shopify_gid`
5. **If: Not migrated** — empty result → Respond `{status:"skip",reason:"not_in_shopify"}`
6. **Build Metafields** (Code node) — fetches enrichment attrs inline via `fetch()`, merges with WC product attrs, applies `SHOPIFY_KEY_OVERRIDES` + `WC_ATTR_ID_TO_SLUG`, returns `{shopifyGID, metafields[], count}`
7. **GraphQL Push** — POST `metafieldsSet` mutation with `variables.metafields = {{$json.metafields}}`
8. **Respond** — `{status:"ok", product, pushed, errors}`

**Key metafield types in use:**
- `single_line_text_field` — 170 of 172 attrs (default for both shopify.* and custom.*)
- `list.single_line_text_field` — `pa_features` → `shopify.features`, `pa_lighting` → `shopify.lighting-features`, `pa_motherboard-form-factor` → `custom.compatible-motherboard-form-factors` (dual-write list)

**Architecture (as built):** All Supabase access uses direct Postgres connection (credential `Supabase Postgres`, `BSoGuZ9BOv4OWqXf`) — the n8n Docker container cannot reach `supabase.reganmcgregor.com.au` via HTTP. Two Postgres nodes:
1. `Get Product` — LEFT JOIN `tl_wc_products_mirror` + `tl_shopify_products_mirror`, always returns 1 row; `shopify_gid` is null if not yet migrated.
2. `Get Enrichment Attrs` — queries `tl_product_attributes` with `alwaysOutputData: true` to avoid 0-item propagation stopping the Code node.

**Shopify type conflicts (resolved 2026-06-15):** 25 `custom.*` metafield definitions had `number_integer` or `dimension` types from Phase 2, conflicting with our `single_line_text_field` mapping. Resolved by querying `metafieldDefinitions(ownerType: PRODUCT)` via a temp n8n workflow and calling `metafieldDefinitionDelete(deleteAllAssociatedMetafields: true)` on each. All 25 deleted; definitions will be lazy-recreated as `single_line_text_field` on next push.

**APP_NOT_AUTHORIZED — `shopify.motherboard-form-factor` (resolved 2026-06-15):** Shopify blocks writes to `shopify.motherboard-form-factor` for PC Cases (wrong product category). Resolved with a **dual-write pattern**: `shopify.motherboard-form-factor` (`single_line_text_field`) stays in the mapping for actual Motherboard products; a second entry `custom.compatible-motherboard-form-factors` (`list.single_line_text_field`) is always injected alongside it for all products. Motherboards get both; PC Cases get only the `custom.*` list (the `shopify.*` attempt returns APP_NOT_AUTHORIZED in `userErrors`, handled gracefully). Also fixed a related enrichment accumulation bug: multiple enrichment rows for the same attribute were previously clobbered (last row won); now all values are accumulated and joined. Final verified test (Corsair 4000D): `pushed: 11, errors: [{APP_NOT_AUTHORIZED on shopify.motherboard-form-factor}]`.

**n8n gotchas (carry forward from Phase 9/10):**
- `}}` / `{{` brace runs in GraphQL strings break n8n's expression parser — space out all braces
- Shopify product/variant ids overflow `integer` — use `bigint` / text for any id column
- `patchNodeField` mangles `$` in replacement strings — use `updateNode` with complete parameters for any value containing `$`
- Deactivate + reactivate after edits to re-register the Schedule Trigger (this workflow is webhook-triggered, so save-while-active re-registers the webhook)

### Batch script

**File:** `~/Developer/techloop-attribute-audit/backfill-phase11a.js`

**What:** Queries all `tl_wc_products_mirror` rows with `acf_supplier_product_id IS NOT NULL`, POSTs each `wc_id` to the Pusher webhook. Resume-on-failure via `backfill-progress.json`.

**Run:**
```
cd ~/Developer/techloop-attribute-audit
node --env-file=/Users/regan.mcgregor/Obsidian/TechLoop/.env backfill-phase11a.js
```

**Actual result (2026-06-17):** 992 products processed — pushed: 992, skipped: 0, errors: 0. Took 168 seconds at concurrency 8. Progress saved to `backfill-progress.json`.

### Post-run audit

**Script:** `~/Developer/techloop-attribute-audit/output/audit-coverage.js`

**Run:** `node --env-file=/Users/regan.mcgregor/Obsidian/TechLoop/.env output/audit-coverage.js`

**Result (2026-06-17):** Written to Google Sheet `1rbJnCrwZJPeXKjzMSzeFDcS5VshLkG16EDJMsWwGQpY` — tabs "11A Shopify Actual" and "11A Coverage".

**Coverage highlights:**

| Category | Products in Shopify | shopify.* coverage |
|---|---|---|
| Desktop Memory | 214 | 97% |
| Laptop Memory | 14 | 86% |
| Network Cables | 70 | 80% |
| AMD Socket AM5 | 17 | 76% |
| CPUs / Processors | 65 | 57% |
| Monitors | 24 | 50% |
| Keyboards | 17 | 35% |
| Laptops | 66 | 20% |
| GPUs | 80 | 21% (Shopify Magic auto-set, not our Pusher) |
| Computer Cases | 187 | 0% |
| SSDs, Fans, Electronics, Accessories | various | 0% |

**Why 0% on Cases/SSDs/Fans/GPUs:** These categories have mostly `pending` attributes in `tl_product_attributes` — the enrichment pipeline was frozen, so no approved data to push. Phase 11.B re-activating the pipeline will resolve this over time.

**Computer Cases note:** `pa_motherboard-form-factor` IS approved for 87 case products and the Pusher attempts `shopify.motherboard-form-factor`, but Shopify rejects it with `INVALID_VALUE` ("Owner subtype does not match") — the attribute is only valid for Motherboard-category products, not Computer Cases. Cases will get shopify.* coverage once `pa_colour` and `pa_case-windowed` are enriched and approved via Phase 11.B.

**normalizeValue fix (2026-06-17):** Added `motherboard-form-factor` normalisation to `TL_Attribute_Pusher_Shopify` Code node — `microATX` → `Micro-ATX`, `E-ATX` → `Extended ATX`. Benefits actual Motherboard products. Computer Cases re-run: 194/194 pushed, 0 errors.

**Smart collections:** All ~20 degraded smart collections (RGB Memory, Black Keyboards, Mechanical Keyboards, etc.) auto-populated from the new metafield values after the backfill — no manual step required.

---

## Phase 11.B — Ongoing Enrichment Rebuild

**What:** Re-point the live enrichment chain at Shopify. Fix outstanding Phase 10 loose ends.

### Reactivate frozen upstream workflows

All were frozen at the 2026-05-05 blanket freeze. Each needs a verify step (feed source / credentials / schedule still valid) before activating.

| Workflow | n8n ID | Reactivate notes |
|---|---|---|
| TL_URL_Discoverer | (WC original) | Verify discovery source still valid |
| TL_Scraper | (WC original) | Verify scrape targets + credentials |
| TL_Enrich_Attributes | (WC original) | Verify Ollama endpoint + Qdrant |
| TL_Attribute_Proposer | (WC original) | Verify Slack channel + credentials |
| TL_Enrichment_Reviewer | (WC original) | Verify HITL Slack flow |

### Re-point HITL approve button at Shopify

The `TL_Slack_Interaction_Handler_Shopify` Handler's `map_to_existing` branch currently targets WooCommerce. Update to call `TL_Attribute_Pusher_Shopify` webhook instead (passing `wc_id` of the approved product).

The malformed Switch connection on `map_to_existing` (left from Phase 10) must be fixed before this button is usable.

### Prompt-tune Publisher AI optimiser

The `TL_Product_Publisher_Shopify` AI title/description node hallucinated a model number from a Qdrant example during Phase 10 verification. Tune the prompt to: ground output strictly in the provided product data, explicitly disallow inventing model numbers not present in the input, and add a validation step comparing the output model number against `leader_product_code`.

---

## Phase 11.C — Targeted Scrape for Gaps (optional)

**Trigger:** Only proceed if the Phase 11.A audit Sheet shows meaningful `shopify.*` coverage gaps worth filling (e.g. whole categories with <30% coverage).

**What:** A scoped targeted scrape pass for thin-coverage categories, feeding new values through the existing enrichment chain (Scraper → LLM → HITL review → `TL_Attribute_Pusher_Shopify`).

**Scope:** TBD after audit Sheet review.

---

## Workflows

| Workflow | Sub-phase | Status | n8n ID |
|---|---|---|---|
| TL_Attribute_Pusher_Shopify | 11.A + 11.B | **Active** (webhook) — Supabase-backed mapping, handles WC-backed + Shopify-native attrs | `s06MMF8cRDi0DXhj` |
| TL_URL_Discoverer | 11.B | **Active** (reactivated 2026-06-18, LIMIT 50) | `QQMwVzR5c79Nx3RL` |
| TL_Scraper | 11.B | **Active** (reactivated 2026-06-18) | `b4rR1gwQ25uUJkEQ` |
| TL_Enrich_Attributes | 11.B | **Active** `*/15 * * * *` (reactivated 2026-06-18) — Shopify-vocab-aware, writes `auto_approved` | `lq8960K0xVwF3Xst` |
| TL_Attribute_Proposer | 11.B | **Active** (reactivated 2026-06-18) | `hUGA1KFWBTGU8K6T` |
| TL_Enrichment_Reviewer | 11.B | **Active** (reactivated 2026-06-18) — surfaces `pending` + `auto_approved` | `OarmHP6DpJvAV0Pb` |

---

## Prerequisites

- Phase 11.0 mapping audit complete (SHOPIFY_KEY_OVERRIDES locked) before building `TL_Attribute_Pusher_Shopify`
- `tl_wc_products_mirror` + enrichment tables accessible in Supabase (verify — WC instance doesn't need to be running, Supabase is the source)
- `TL_Attribute_Pusher_Shopify` built + tested before Phase 11.B HITL re-pointing

## Open questions / follow-ups

- WC decommission — **Done in [[Phase 13 — WooCommerce Decommission]] (2026-06-18)**. All Phase 11.B workflows verified live against Shopify; WC instance and artefacts fully removed.
- Phase 11.C scope TBD after audit Sheet

---

## Definition of Done — Achieved (2026-06-18)

All sub-phases complete. Key outcomes:

- 11.0: 335-attr mapping locked (170 WC-backed + 165 Shopify-native) in `tl_attribute_mapping`; vocab seeded in `tl_shopify_metaobject_values` (461 rows).
- 11.A: 992 products backfilled to Shopify metafields. Smart collections auto-populated.
- 11.B: 5 enrichment workflows reactivated; `auto_approved` safety gate in place; Pusher Supabase-backed; Handler `approve_attr_mapping` fires Pusher; Publisher AI prompt-tuned to prevent model-number hallucination. Review queue cleared (200 approved, 14 rejected).

Deferred to Phase 13 or later: `list.single_line_text_field` Shopify-native attrs (bag-case-features etc.); `motherboard-form-factor` dual-write via `fan_out`; category-applicability gating; `TL_Sync_Shopify_Attribute_Vocab` n8n wrapper; `tl_shopify_metafield_definitions_mirror` population.
