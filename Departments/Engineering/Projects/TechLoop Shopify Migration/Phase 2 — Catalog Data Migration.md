---
status: done
notion_page_id: 3568c3d3-c03e-8142-93d9-f775890bca5c
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-19T15:49:26Z
---

# Phase 2 — Catalog Data Migration

## Context

Snapshot WC products from `tl_wc_products_mirror`, transform per [[Phase 1 — Catalog Model]] mappings, and bulk-import to Shopify via Admin GraphQL. Create smart collection rules, populate multi-location inventory from the Leader feed snapshot. Parallelised by category bucket under orchestrator coordination. Marked Done in Notion on 2026-05-09.

## Reported outcome (2026-05-09)

Final catalog state on the live Shopify store: 1,209 DRAFT products, 1,208 with images (99.92%, 1 deferred to Phase 9 Crawl4AI scrape), 52 Brand metaobjects + Micron parent, 155+ product metafield definitions, 28 smart collections, NSW-only inventory (711 products with stock, 10,962 units total). The 16-commit chain that lands Phase 2 spans `2562380` → `9900f71`.

## Live-verified state (2026-05-20)

Re-queried via Shopify CLI on 2026-05-20. **Drift observed:**

| Item | Step 04–07 reported | Live now | Notes |
|---|---|---|---|
| Product count | 1,209 | 1,209 | ✓ |
| Product status | DRAFT (Step 05 body) | **ACTIVE** | Drift — bulk-activated since Phase 2 closed. Storefront still password-gated via Online Store → Preferences (not API-verifiable). |
| Smart collections | 28 (Step 06 final) | **266** | Drift — likely series-style smart collections added post-Step 06 (e.g., Vengeance LPX, Vengeance Pro, Vengeance RGB Pro, etc.) |
| Manual collections | (implied small) | 71 | — |
| **Total collections** | 280 | **337** | Drift — needs reconciliation against vault `Phase 1 — Catalog Model.md` 2026-05-17 dedupe (which ended at ~334) |
| Brand metaobjects | 52 + Micron parent | 52 (type `brand`) | ✓ — Micron must be one of the 52 |
| Locations | 5 | 5 (NSW, QLD, SA, VIC, WA) | ✓ |
| Ubiquiti UniFi G5 Turret Ultra image | deferred | mediaCount: 0 | ✓ — still deferred |

The metafield schema on a spot-checked product (Ubiquiti UniFi Protect NVR Instant Kit) shows the expected shape: `custom.brand` metaobject_reference, `custom.short_description` rich_text_field, all 7 `custom.supplier_*` fields populated, and `custom.length` / `.width` / `.height` as `dimension` type with CENTIMETERS unit.

## Step 04 — 10-product sample import + schema upgrade

Import a 10-product sample spanning all major categories to Shopify via Admin GraphQL. Validate that metaobject references, brand assignments, category mappings, and metafield definitions are correctly applied. Controlled test before full import.

**Single-location-NSW decision** (commit `2bff183`): Leader feed only has aggregate stock (`availability_total`), not per-state. All stock now goes to the NSW Shopify Location; other 4 locations stay structurally in place but with no inventory records. Re-run of step 04 successfully updated all 10 products' inventory at NSW.

**Field gap fixes (2026-05-08)**:
- Commit `910d319`: added price (regular_price → variant.price; sale_price → compareAtPrice when active), SKU, barcode (acf_supplier_barcode), inventoryItem.cost (wc_cog_cost), variant weight + dimensions (fetched from WC REST API per product), long description (from WC API into descriptionHtml). Added `custom.short_description` (multi_line_text), `custom.mpn` (single_line_text), `custom.dimensions` (single_line_text "L × W × H cm") metafield definitions.
- Commit `d718811`: added `custom.supplier_cost_inc_gst` (number_decimal, storefront-private) for in-admin margin reporting; populated from acf_supplier_dbp_inc_gst.

**Schema upgrade (commit `9a2628f`)**:
- New supplier-data layer: 7 prefixed metafields (`custom.supplier_name`, `supplier_product_id`, `supplier_status`, `supplier_mpn` (renamed from `custom.mpn`), `supplier_gtin`, `supplier_cost_ex_gst`, `supplier_cost_inc_gst`). All storefront-private except `supplier_mpn` which is public for Google Shopping feeds.
- `variant.barcode` source switched from `acf_supplier_barcode` to `global_unique_id` (WC native GTIN — populated on 1,206/1,209 products). Supplier-reported barcode preserved separately in `custom.supplier_gtin` (35 products differ between authoritative GTIN and supplier's reported value).
- `inventoryItem.cost` switched from ex-GST to inc-GST (`cogs` column) so Shopify margin reports compute correctly against inc-GST prices.
- `custom.short_description` recreated as `rich_text_field` (was multi_line_text); HTML bullet lists now render as proper Shopify list nodes.
- `custom.dimensions` (single text) split into `custom.length` / `.width` / `.height` — each Shopify `dimension` type with CENTIMETERS unit.
- New script `src/steps/04a-metafield-definitions-ddl.ts` handles definition deletes/recreates idempotently.

**No-match category re-audit (commit `b398835`)**: re-audited the 9 no-match WC categories from 1a; 5 promoted to direct match (Motherboard Accessories → `el-7-9-10`, Smart Lighting → `hg-13-8`, Smart Power → `el-7-15-8`, Operating Systems → `so-1-13`, Powerline Networking → `el-12-6-3`); Smart Home Automation kept as catch-all with per-product overrides; Electronics (58 products) deferred to step 05 second-pass enrichment. New file `data/per-product-category-overrides.json` maps wc_id → Shopify category GID for catch-all cases. TP-Link Tapo (wc_id 26015) now correctly assigned `hg-2-3` (Motion Sensors).

**Type promotions (commit `f068906`)**: 14 integer + 7 dimension metafield type promotions (drive bays / port counts / threads / m.2 slots → `number_integer`; cable-length / cord-length → `dimension` METERS; fan-size / max-cpu-cooler-height / max-gpu-length / max-psu-length / driver → `dimension` MILLIMETERS). Step 04 transform now dispatches on metafield definition type — emits `{value, unit}` JSON for dimension fields, parsed integers for integer fields.

**Standard Taxonomy mapper (commit `ae02904`)**:
- Built `src/lib/taxonomy.ts` with 5-pass fuzzy matcher (exact → normalised → starts-with → contains → substring) returning Shopify `TaxonomyValue` GIDs from WC term value names.
- Step 04 startup: loads 3,310 canonical taxonomy attributes from `/Developer/product-taxonomy/dist/en/attributes.json`, fetches existing `shopify.*` metafield definitions (9 found, lazily auto-created by Shopify per category), caches metaobject GIDs per type.
- For each keep-standard 1c row on a product: resolves workbook handle → actual `shopify.*` key (via `SHOPIFY_KEY_OVERRIDES` map for known mismatches like `memory-storage-type → memory-technology`, `color → color-pattern`), maps WC term values to TaxonomyValue GIDs, creates missing taxonomy metaobjects on demand, emits `shopify.<handle> = list.metaobject_reference` value.

**Refactor (commit `439a8b7`)**: all transform logic moved to `src/lib/product-transform.ts` shared library. Step 04 itself became a thin wrapper (~130 lines vs ~2,290). Single source of truth for the transform pipeline; unblocked Step 05.

Step 04 commit chain: `2562380 → 2bff183 → 910d319 → d718811 → 9a2628f → b398835 → f068906 → ae02904 → 439a8b7`. Idempotent re-runs produce 0 changes.

## Step 05 — Full 1,209-product import

Import the complete product catalog to Shopify using Admin GraphQL bulk operations (`productCreate` / `productUpdate` mutations). Monitor for errors, retry failed records, validate import counts.

**Completed 2026-05-08.** Final tally: 1,152 created + 8 updated + 20 skipped (resume logic) + 29 retried after category-scoped metafield bug fix = **1,209 products as DRAFT** (later bulk-activated — see live-verified state above). Total elapsed: ~1h 45min for initial run + 109s for retry. NAS image cache: 93 MB across ~858 product folders. Bug fix commits: `4282dfa` (filter `shopify.*` by category supported attributes), `8bdfa6d` (retry pass — all green).

**Image redo (2026-05-09, commits `ca88797`, `ef2a672`)**:
- User direction: **never use Leader supplier feed for product images** (raw/unedited). WC gallery is the only image source.
- Code: removed Leader-feed fallback entirely. For each product, delete all existing Shopify media first, then upload only WC gallery images. Products with empty WC gallery flagged in `data/no-wc-images-2026-05-08.txt`.
- Backfill: 1,209 products processed in 5h 20min, replacing previous Leader-feed images with WC gallery (delete-and-replace). 6 transient infrastructure failures (Shopify 502, fetch failed, NAS SMB blip) — all 6 succeeded on targeted retry.
- Final image coverage: **1,208 / 1,209 = 99.92%.** NAS image cache: 298 MB.
- Outstanding: 1 product (wc_id 24403, Ubiquiti UniFi G5 Turret Ultra 2K HD PoE Camera) has empty WC gallery — deferred to Phase 9 enrichment via Crawl4AI scrape from `ui.com` manufacturer page.

## Step 06 — Smart collections + empty-category backfill

Create smart collection rules per Phase 1 design: 15 from category merges (with attribute filters for `processor-socket`, `recommended-use`, `compatible-device`, `memory-form-factor`) and 16 from WC tags (DDR4, Gaming Memory, CORSAIR, RGB Memory, Mechanical Keyboards, etc.).

**Completed 2026-05-09 (commit `46e36b4`).** 28 smart collections net (13 from 1a merges + 14 from 1d tag rows + 1 curated "Gaming Memory"). Top by population at the time: Laptop Memory SO-DIMM (229), RGB Memory (229), DDR4 RAM (197), Corsair RAM (178), Gaming Keyboards (61). Idempotent re-run = 0 changes.

**20 known rule fallbacks (graceful degradation)** — collections degrade to category-only rules where Shopify's `shopify.*` metafield definitions don't exist yet OR metaobject values haven't been populated:
- 0-product collections: AMD AM4 Motherboards, AMD AM5 Motherboards, Black Keyboards, etc — rules correct but no products have the targeting metafield value set
- Over-inclusive: SO-DIMM and RGB Memory both showed 229 (all RAM) — falls back to category-only when memory-form-factor/lighting metafields/values missing
- Missing `shopify.*` definitions (Shopify creates lazily): `recommended-use`, `compatible-devices`, `keyboard-type`, `keyboard-backlight-type`, `keycap-profile`

These will refine as products get enriched in Phases 9–12 (rebuilt n8n workflows + manual category metafield acceptance per the Shopify Category AI suggestions limitation). Not blocking for storefront launch — collections function, just over- or under-include until data fills in.

**Empty category backfill + cleanup (commits `45f4ac8`, `5dbd579`, `2fdbc2c`, `9f54db4`)**:
- Backfilled all 286 WC categories as Shopify collections (52 already covered + 234 new = 36 clean smart + 12 ambiguous + 186 NO_MATCH manual placeholders).
- Applied 9 user-specified mappings to ambiguous/wrong collections (PC Gaming Headsets, Camera Accessories, Wireless/Wired Networking Adapters, Connected Home, Gaming, PS4, Xbox One, Nintendo Switch — 3 with vendor filter for console smart collections).
- User reviewed the 172 NO_MATCH options (deduped from 186) via Google Sheets workbook (https://docs.google.com/spreadsheets/d/1hZFT0E5GKYfaEqm4n_RRu04bU6OBbjSpVVQ1PRBKKxU/edit). Outcomes: 91 promoted to smart collections (88 delete+recreate, 3 in-place updates), 32 kept manual, 13 kept umbrella (PC Components, Memory, Gaming, etc), 34 deleted (Esports, Works-with-X tags, etc), 5 unresolved/blank (manual follow-up TBD).
- Final Shopify collection count at Step 06 close: **280** (down from peak 314 after deletions).
- 5 collections with unresolved decisions worth revisiting: **Gaming Eyewear, Party Speakers, Memory (RAM) umbrella, Intel Socket 1851, Camera Mounts/Grips/Stabilizers** (all verified still present on live store 2026-05-20).

## Step 07 — Multi-location inventory populate

Ingest Shopify inventory levels from the Leader feed snapshot for each location. Per single-NSW decision (Step 04), all inventory goes to NSW; other 4 locations remain at zero.

**Completed 2026-05-09 (commit `9900f71`).** 1,209 products audited. 872 with `supplier_product_id` (present in Leader feed), 337 without (no feed row, inventory left as-is). Stock > 0: **711 products, total NSW stock = 10,962 units.** Drift: 0 — step 04/05 snapshot still current; no writes needed on full run. Idempotency confirmed. Single-location NSW pattern locked in until Phase 9 rebuilt inventory sync workflow comes online.

## Handoff to Phase 3 (status conflict)

Phase 2 Notion body says "Awaiting user QA pass before Phase 3 (pages, redirects, SEO plumbing)." Phase 3 task currently shows "Not Started" in Notion. The Phase 5 (Hyper) spec body asserts "Phase 3 (pages, redirects, SEO) — Done — content already exists." Conflict to resolve.

Also worth noting before Phase 3:
- Products are now ACTIVE — needs investigation: was this part of intended workflow or a separate activation? Verify password-gate still enabled before any further public-facing change.
- Collection count drift: live 337 vs Step 06 close 280 — reconcile.
