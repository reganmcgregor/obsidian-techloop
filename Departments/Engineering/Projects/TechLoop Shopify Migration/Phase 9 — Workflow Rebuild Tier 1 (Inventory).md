---
status: done
notion_page_id: 3568c3d3-c03e-81aa-b66a-dfe674300985
notion_parent_id: 3558c3d3-c03e-8164-9b2c-ff4243c5147a
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-30T12:00:00Z
---

# Phase 9 — Workflow Rebuild Tier 1 (Inventory)

First tier of n8n workflow rebuilds after the WC→Shopify cutover. Re-point the inventory-critical workflows at Shopify.

**Scope (authoritative, per Notion spec):** rebuild **two** workflows —
1. **TL_Mirror_Shopify** — replaces `TL_Mirror_WooCommerce`; keeps the Supabase mirror current. ✅ **Done.**
2. **TL_Inventory_Syncer_Shopify** — Leader feed → Shopify inventory + cost + supplier metafields. ✅ **Live.**

> **Scope correction:** `TL_Product_Detector` is **Phase 10 (Onboarding)**, NOT Phase 9 — the meta-plan's summary table wrongly lumped it in. Phase 9 = Mirror + Syncer only. (The `vw_new_products_for_shopify_onboarding` view already created is harmless Phase-10 prep.)

The Supabase mirror is the integration boundary: only the Mirror's source-pull and the Syncer's target-push are Shopify-specific; the state machine, batching, Slack, and execution logging are platform-agnostic and ported from the FROZEN WC originals (clone-and-modify; FROZEN kept for rollback).

---

## Status

| Workflow | Status | n8n id | Schedule |
|---|---|---|---|
| TL_Mirror_Shopify | ✅ Built, tested, **live** | `B6BBVJkRRCZP69HW` | `15 */4 * * *` |
| TL_Inventory_Syncer_Shopify | ✅ Built, **tested live (5/5 verified)**, full sync run + schedule active | `uQ6XmGi3mspxKD56` | `30 */3 * * *` |

**Mirror verification (2026-05-30):** first-run full sync mirrored **1,209 products / 1,209 variants / 1,209 NSW inventory levels**; 872 with `supplier_product_id`, 868 with unit cost; incremental run + Slack Block Kit + `tl_workflow_executions` logging all pass.

**Syncer verification (2026-05-30):** diff SQL dry-run validated (joins correct); all 3 write mutations (`inventorySetQuantities` + `inventoryItemUpdate` + `metafieldsSet`) executed cleanly via a single-product no-op probe (all `userErrors: []`). Built but **inactive**.

> ## ✅ GO-LIVE BLOCKER — RESOLVED 2026-05-30
> Resolved: `TL_Ingest_Leader_Feed` reactivated (feed fresh — 19,628 live products), controlled 5-product test verified 5/5 on live Shopify, full sync run (387 synced, 0 failed), schedule activated. Original blocker for reference:
> `tl_feed_leader_raw` last refreshed **2026-05-03 (~27 days stale)** — `TL_Ingest_Leader_Feed` has been frozen since before cutover. With a stale feed, every product trips the 6-hour `stale` rule and the Syncer would **zero all live inventory**. A **feed-freshness guard** was added to the diff (`AND (SELECT max(last_seen_at) FROM tl_feed_leader_raw) > NOW() - INTERVAL '12 hours'`) so the Syncer safely returns 0 rows while the feed is stale — confirmed (0 actionable rows now).
> **Go-live procedure:** (1) reactivate `TL_Ingest_Leader_Feed`, verify a fresh pull lands in `tl_feed_leader_raw`; (2) re-run the diff dry-run — expect real `stock_only`/`cost_only`/`stock_and_cost` rows; (3) controlled Syncer test via the temp webhook with `LIMIT 5`, read back a few products on Shopify; (4) restore `LIMIT 500`, remove the temp webhook, deactivate→reactivate to register the schedule.

---

## Foundations laid (done, shared by both workflows)

**Auth — client-credentials, NOT the n8n Shopify OAuth2 credential.** The store app is a Dev Dashboard app; its OAuth tokens expire in minutes with no refresh, so the n8n `Shopify OAuth2 API` credential is unusable for scheduled runs. Working pattern: a `Get Token` HTTP node POSTs `…/admin/oauth/access_token` (form-urlencoded, `grant_type=client_credentials`) using n8n **Custom Auth credential `Shopify Client Credentials` (`4Kf8ZBknzZl6bvvi`)** which injects the client id/secret; returns a 24h token. Downstream GraphQL nodes set `X-Shopify-Access-Token: {{ $('Get Token').first().json.access_token }}`. App = "Shopify n8n App" (all scopes; `atkn_` automation token is CI/CD-only and unused).

**DDL (Supabase, applied):**
- `tl_shopify_products_mirror` + supplier columns (`supplier_name`, `supplier_product_id`, `supplier_status`, `supplier_mpn`, `supplier_gtin`, `supplier_cost_ex_gst`, `supplier_cost_inc_gst`) + index.
- `tl_shopify_variants_mirror` + `unit_cost`.
- `tl_product_supplier_status` re-keyed on `(supplier_product_id, supplier_name)` (was `wc_id`); 5 duplicate WC-draft pairs collapsed → 966 rows.
- `tl_shopify_locations_mirror` seeded with the 5 AU locations (FK target for inventory levels).

**Shopify facts:** store admin host `techloop-7.myshopify.com` (API `2025-07`); NSW location GID `gid://shopify/Location/82672910534` (the only `shipsInventory` location). Single-location-NSW model.

---

## Build learnings (apply to the Syncer)

- **GraphQL `}}`/`{{` brace runs break n8n's expression parser** — space out all braces in query strings (GraphQL is whitespace-insensitive).
- **Query cost ceiling:** `products(first: 25 …)` heavy query = requestedQueryCost 650 (under the 1000 limit; `first:50` rejected). Cursor pagination via HTTP-node `updateAParameterInEachRequest` + `requestInterval: 750` respects the 2000-pt/100-per-s throttle bucket.
- **One Postgres node can run all 3 mirror upserts** (multi-statement, `$smir$`-dollar-quoted `jsonb_array_elements`, `=`-expression prefix for the embedded `{{ JSON.stringify(...) }}`).
- **`inventorySetOnHandQuantities` is deprecated** → use `inventorySetQuantities(input:{ name:"on_hand", reason, ignoreCompareQuantity:true, quantities:[{inventoryItemId, locationId, quantity}] })`.
- **`read_markets_home` scope** is required by the product/inventory read query (easy to miss).
- Build n8n workflows incrementally via `addNode` partial updates (large single-create payloads can drop); validate with `n8n_validate_workflow`; test schedule-triggered workflows via a temporary webhook trigger (removed before go-live); after edits, deactivate+reactivate to re-register the Schedule Trigger.

---

## TL_Inventory_Syncer_Shopify — design (next)

Clone FROZEN `TL_Inventory_Syncer` (`ZgqlMhNqh3TJMB6G`). Preserve the supplier-status state machine (active/missing/discontinued/stale, 7-day promotion), SplitInBatches(10)+Wait(2s), Slack, logging. Swap:
- **Calculate Diff SQL** — join `tl_shopify_products_mirror ↔ _variants_mirror ↔ _inventory_levels_mirror (NSW)` ↔ `tl_feed_leader_raw` on `supplier_product_id`; compare `on_hand` vs leader stock, `unit_cost` vs leader cost; status CTE keyed on `supplier_product_id`.
- **Push** — `Get Token` → per actionable row emit `inventorySetQuantities` (stock), `inventoryItemUpdate` (cost), `metafieldsSet` (`custom.supplier_status` + cost metafields).
- **Status upsert** — `tl_product_supplier_status` on `(supplier_product_id, supplier_name)`.

Test one product per `change_type` against the live store (read-back) before activating writes.

---

## Out of scope / follow-ups

- Single Ubiquiti product (wc_id 24403) with empty gallery — tagged "Phase 9 Crawl4AI enrichment" in the meta-plan but is really enrichment (Phase 11), not inventory.
- Mirror Slack posts to `#wc-mirror` (`C0ADR038U6Q`) for now — rename to a Shopify channel if desired.
- Mirror query trimmed to inventory-essential fields (no SEO/tags/weight) — extend if the Publisher tier needs richer mirror data.
- Domain: `techloop-7.myshopify.com` works for token-based Admin API calls; the *permanent* domain is `zc30tg-fi.myshopify.com` (the meta-plan recommends it for OAuth-callback flows, which we don't use). Consider switching workflow URLs to the permanent domain for durability.
