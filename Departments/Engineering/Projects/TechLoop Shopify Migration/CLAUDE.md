# CLAUDE.md — TechLoop Shopify Migration

Project-scoped guidance for the WC→Shopify migration. Complements (does not replace) the vault-root `CLAUDE.md`. Keep this lean — operational keys + gotchas, not full specs (those live in the `Phase N — …` docs).

## What this is
Migrating techloop.com.au from self-hosted WooCommerce to **Shopify Basic**. DNS flipped 2026-05-24 (Phase 8). Now rebuilding the n8n automation suite against Shopify (Phases 9–12). Status table + phase docs: [[TechLoop Shopify Migration]].

## Phase status (one-liner)
- Phases 0–8: **Done** (catalog, redirects, customers, theme, channels, shipping/payments, cutover).
- **Phase 9 (Inventory) — ✅ Done (2026-05-30):** TL_Mirror_Shopify + TL_Inventory_Syncer_Shopify both **live**. Syncer first full run synced 387 products (238 corrections + 149 zero-outs), 0 failed. Scope = Mirror + Syncer only.
- **Phase 10 (Onboarding) — ✅ Done (2026-05-31):** Detector (`fz50fq1dCmrHhazX`, active) + Publisher (`pgWWBz9f6RXLXEIe`, built+verified, **inactive** pending user go-live) + Slack Handler (`hikIeVV081e76pEv`, active). Category map fully resolved (426 approved / 0 pending); onboarding view yields 771 products. Only remaining = flip Publisher active. See [[Phase 10 — Workflow Rebuild Tier 2 (Onboarding)]].
- **Phase 11 (Enrichment) — ✅ Done (2026-06-18):** 11.A bulk backfill (992 products) + 11.B chain fully reactivated. Supabase-backed attr mapping (335 attrs = 170 WC-backed + 165 Shopify-native; `tl_attribute_mapping` + `tl_shopify_metaobject_values`); `auto_approved` safety gate (never auto-pushes to live store); Pusher handles WC-backed + Shopify-native attrs with correct namespace; 5 enrichment workflows live; review queue cleared (200 approved, 14 rejected).
- **Phase 12 (Alerts/Utilities) — ✅ Done (2026-06-18):** TL_Price_Watchdog (`yExNIbiIGVFOoJ4N`) rebuilt for Shopify + "Set to RRP" Slack button. TL_Queue_Reviewer_Shopify (`20i0wyIaAnu9h2Wh`) already active.
- **Phase 13 (WC Decommission) — ✅ Done (2026-06-18) — CLOSED:** WP/WooCommerce instance fully decommissioned. Dropped 5 WC mirror tables (backed up to `_archive/phase13-wc-decom/`); renamed `wc_product_id → shopify_product_id` across 3 tables; rewrote 12 views WC-free; deleted 5 n8n workflows (4 frozen WC originals + TL_Attribute_Proposer); edited 5 active workflows to remove remaining WC nodes. End-to-end verified (Enrich exec 57884 + Pusher exec 57890). Final read-only sweep (2026-06-18) confirmed zero functional WC references across all 13 active workflows and Supabase schema — only cosmetic WC-named identifiers remain (see Phase 13 doc). See [[Phase 13 — WooCommerce Decommission]].
- **Phase 14 (Enrichment Service Rebuild) — Not Started (parked):** Complete ground-up rebuild of the enrichment → Shopify metafield pipeline. WC and Shopify attribute models are not like-for-like; the migrated chain carries WC-era assumptions. Design inputs captured. See [[Phase 14 — Enrichment Service Rebuild]].

## Shopify store
- **Admin API host:** `techloop-7.myshopify.com` (works for token-based Admin GraphQL; API version `2025-07`). Permanent domain is `zc30tg-fi.myshopify.com` — required for **OAuth-callback** flows (we don't use those); fine to prefer it for durability.
- **Storefront:** `www.techloop.com.au`. App: "Shopify n8n App" (Dev Dashboard, all scopes).
- **Single-location-NSW** inventory model. NSW location GID `gid://shopify/Location/82672910534` (only `shipsInventory` location). Others: QLD `82873581766`, SA `82873680070`, VIC `82873614534`, WA `82873647302`.

## n8n ↔ Shopify auth (IMPORTANT)
- The native n8n **Shopify node is REST-only** — use **HTTP Request + Admin GraphQL** for everything (inventory levels, metafields, unit cost, metaobjects).
- The n8n **`Shopify OAuth2 API` credential does NOT work** (Dev Dashboard tokens expire in minutes, no refresh).
- **Working auth:** each workflow's first node is a `Get Token` HTTP node → POST `https://techloop-7.myshopify.com/admin/oauth/access_token` (form-urlencoded, `grant_type=client_credentials`), Generic Credential Type → **Custom Auth cred `Shopify Client Credentials` (`4Kf8ZBknzZl6bvvi`)** which injects client_id/secret. Returns a 24h token. Downstream GraphQL nodes header `X-Shopify-Access-Token: {{ $('Get Token').first().json.access_token }}`.
- The app's `atkn_` "app automation token" is **CI/CD deploy only** — not an API token.

## Workflows (n8n ids)
- **TL_Mirror_Shopify** `B6BBVJkRRCZP69HW` — live, schedule `15 */4 * * *`.
- **TL_Inventory_Syncer_Shopify** `uQ6XmGi3mspxKD56` — **live**, `30 */3 * * *`. Writes stock/cost/supplier_status to Shopify via aliased GraphQL mutations. Diff has a feed-staleness guard (no-ops if the whole feed is >12h stale).
- **TL_Ingest_Leader_Feed** `mUkhS0BfN7Lq6EsS` — ✅ reactivated 2026-05-30 (every 2h, feed fresh).
- **TL_URL_Discoverer** `QQMwVzR5c79Nx3RL` — ✅ reactivated 2026-06-18, LIMIT 50 candidates.
- **TL_Scraper** `b4rR1gwQ25uUJkEQ` — ✅ reactivated 2026-06-18 (Crawl4AI at `http://crawl4ai:11235/crawl`).
- **TL_Enrich_Attributes** `lq8960K0xVwF3Xst` — ✅ reactivated 2026-06-18 (`*/15 * * * *`). Shopify-vocab-aware; writes `auto_approved` (not `approved`) for high-confidence promoted attrs.
- **TL_Enrichment_Reviewer** `OarmHP6DpJvAV0Pb` — ✅ reactivated 2026-06-18. Fetches `pending` + `auto_approved`; shows 🤖 Auto / ⏳ Pending label.
- **TL_Attribute_Pusher_Shopify** `s06MMF8cRDi0DXhj` — webhook-triggered. Reads mapping from `tl_attribute_mapping` + `tl_shopify_metaobject_values`; handles WC-backed attrs (`wc_attribute_id IS NOT NULL`) and Shopify-native attrs (`shopify_key IS NOT NULL`).
- **TL_Slack_Interaction_Handler_Shopify** `hikIeVV081e76pEv` — `approve_attr_mapping` fires Pusher; WC API nodes removed (Phase 13).

## Supabase mirror tables
`tl_shopify_products_mirror` (+ `supplier_*` cols), `tl_shopify_variants_mirror` (+ `unit_cost`), `tl_shopify_inventory_levels_mirror` (FK `location_id → tl_shopify_locations_mirror.shopify_id`, FK `variant_id → variants.shopify_id`), `tl_shopify_locations_mirror` (seeded), `tl_feed_leader_raw`, `tl_product_supplier_status` (re-keyed on `(supplier_product_id, supplier_name)`), `tl_onboarding_queue`, `tl_workflow_executions`. View `vw_new_products_for_shopify_onboarding` (Phase-10 prep; excludes both mirrors).

## n8n + Shopify gotchas
- **GraphQL `}}`/`{{` brace runs break n8n's expression parser** — space out all braces in query strings.
- **Query-cost ceiling 1000:** `products(first: 25 …)` heavy query = 650; `first:50` rejected. Cursor pagination via HTTP-node `updateAParameterInEachRequest` + `requestInterval: 750` for the 2000-pt/100-per-s throttle bucket.
- **`inventorySetOnHandQuantities` deprecated** → `inventorySetQuantities` (`name:"on_hand"`, `ignoreCompareQuantity:true`).
- Product/inventory read query needs scope **`read_markets_home`**.
- Build workflows incrementally via `addNode` (large single-create payloads drop); `n8n_validate_workflow` before testing; test schedule-triggered workflows via a temp webhook trigger; **deactivate+reactivate after edits** to re-register the Schedule Trigger.
- Host has `N8N_BLOCK_ENV_ACCESS_IN_NODE=true` → `$env` in expressions is blocked; keep secrets in n8n credentials, not env vars.

## Conventions
- Clone-and-modify FROZEN workflows (don't mutate the originals; they're the rollback).
- Per-phase docs are `Phase N — …` rows in Notion **Tasks (Engineering)** DB (canonical spec in row body) mirrored to vault docs (`canonical: obsidian`). Update both on progress; tick Status at end-of-step (user runs from Notion mobile).
- Australian English. No `- [ ]` checkboxes in vault (tasks live in Notion).
