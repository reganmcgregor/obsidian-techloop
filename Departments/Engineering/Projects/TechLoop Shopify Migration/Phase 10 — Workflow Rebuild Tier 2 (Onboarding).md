---
status: done
notion_page_id: 3568c3d3-c03e-8132-a3db-f0d184b0d3c5
notion_parent_id: 3558c3d3-c03e-8164-9b2c-ff4243c5147a
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-31T08:00:00Z
---

# Phase 10 — Workflow Rebuild Tier 2 (Onboarding)

Second tier of the n8n rebuild after the WC→Shopify cutover: the **new-product onboarding** chain. Detect genuinely-new Leader products → AI-optimise title/description → human review in Slack → publish to Shopify as a **draft** → one-click go-live. See [[TechLoop Shopify Migration]], [[Phase 9 — Workflow Rebuild Tier 1 (Inventory)]].

**Build complete + verified (2026-05-31).** The only remaining step is the operational go-live (flip the Publisher to active) — a deliberate user trigger, since it starts creating draft products on the live store.

---

## Workflows

| Workflow | Status | n8n id | Schedule / Trigger |
|---|---|---|---|
| TL_Product_Detector_Shopify | ✅ Done, tested, **active** | `fz50fq1dCmrHhazX` | `45 */2 * * *` |
| TL_Product_Publisher_Shopify | ✅ **Active** (activated 2026-06-06) | `pgWWBz9f6RXLXEIe` | `*/5 * * * *` |
| TL_Slack_Interaction_Handler_Shopify | ✅ Built + verified, **active** | `hikIeVV081e76pEv` | webhook `/webhook/slack-interaction` |

All three are clone-and-modify from FROZEN WC originals (Detector ← `rnUjGGsdD1jdaFIg`, Handler ← `rDloLTYACT56kfpn`); FROZEN kept for rollback.

---

## Onboarding flow

```
Detector (every 2h)  → vw_new_products_for_shopify_onboarding → INSERT tl_onboarding_queue (status=pending_review)
                       → Slack approve/reject buttons
Handler (webhook)    → approve → queue status=approved
Publisher (every 5m) → fetch approved → lock → fresh Leader data (+ category map) → AI title/desc/short-desc (Ollama qwen2.5:14b + Qdrant RAG)
                       → price → brand → productSet (DRAFT, full metafield parity) → inventoryActivate (NSW) → mark published
                       → Slack "Publish Now" button
Handler (webhook)    → Publish Now → productUpdate status=ACTIVE + publishablePublish (Online Store) → queue status=live → Slack confirm
```

Image pipeline is fully active: Download → square + white-pad (10%) → cap 2048px → `stagedUploadsCreate` (PUT) → `Merge Upload Data` → `Upload Image to Staged URL` → `Capture Staged Resource URL` → attached via `input.files[].originalSource`. Falls back to raw `leader.image_url` only if download fails.

---

## productSet (2025-07) — verified on live store

- `productSet(synchronous:true, input:ProductSetInput!)` — images go in `input.files:[{originalSource, contentType:'IMAGE', alt}]` (NOT `media:`, that's productCreate).
- New-product inventory can't be set inside productSet → separate `inventoryActivate(inventoryItemId, locationId, available)` after, at NSW location `gid://shopify/Location/82672910534`.
- **Metafield `type` must match the store definition exactly** or productSet throws userErrors. Authoritative types: `custom.short_description` = **rich_text_field** (Shopify rich-text JSON AST, not HTML); `custom.warranty` = single_line_text_field; `custom.supplier_*` = single_line_text_field except `supplier_cost_inc_gst`/`supplier_cost_ex_gst` = number_decimal; product dimensions `custom.length/width/height` = **dimension** (`{value, unit:'MILLIMETERS'}` — Leader dims are mm); variant `mm-google-shopping.condition`='new' + `.mpn` = single_line_text_field.
- Brand/category auto-filing: store has smart collections on **vendor** and **product_type**, so setting `vendor` + `productType` auto-joins the brand + category collections.

## Category mapping (`tl_shopify_category_map`)

Leader `(category_name, subcategory_name)` pairs → Shopify **taxonomy GID** + **breadcrumb collection GID** + **product_type**. Drives node 27's `category`, `productType`, and `breadcrumb.primary_collection` (collection_reference metafield).

- **Round 1:** 314 pairs approved + 2 Cases pairs (services/ambiguous skipped).
- **Round 2 (2026-05-31):** after the merchandising block-flag review (21 categories unblocked), 110 newly-opened pairs were mapped (taxonomy + product_type for all, breadcrumb collection where the store has one) and approved.
- **Final: 426 approved / 8 skip / 0 pending.** Onboarding view yields **771 eligible products**, 100% with taxonomy + product_type.
- **Join key:** trimmed equality on category + subcategory — `regexp_replace(COALESCE(x,''),'^\s+|\s+$','','g')` (handles `\r\n` empty-subcategory rows). The Publisher's `Get Fresh Leader Data` query LEFT JOINs the approved map; the onboarding view INNER JOINs it (so only approved pairs flow).
- No store collection exists for phone cases / phones / screen protectors / UPS / printer / chargers → those onboard with taxonomy + product_type but a NULL breadcrumb (fine). Collections exist for Monitors, Memory(RAM), Motherboards, Optical Drives, Desktops, Mini PCs, Laptops, Network Cables, KVM, Add-On Cards, Bluetooth Trackers.

## Block flags (merchandising gate)

The onboarding view also respects `tl_category_map.is_blocked` / `tl_brand_map.is_blocked`. At the cutover-era state every fresh candidate was blocked. Reviewed via approval Sheet 2026-05-31: **21 categories unblocked** (mainstream consumer/SMB — mobile cases, screen protectors, NUC/SFF, UPS, mobile phones, printer consumables, cables, memory, monitors, motherboards, etc.); kept blocked: VOIP ×3, Servers, Network-Security UTM, Data Racks, spare parts, Commercial Display, Furniture, Software/Licensing, Labelling, Point of Sale. **All 7 brands kept blocked** (VOIP brands redundant with category blocks; Teltonika = enterprise IoT).

---

## Build learnings / gotchas

- **Staged upload uses `httpMethod: 'PUT'`** (not POST multipart): Shopify/GCS returns 7–9 dynamic form parameters for POST — hardcoding the count breaks silently. PUT sends the binary as the body with `Content-Type: image/jpeg` only — no parameters needed.
- **Binary lost after `stagedUploadsCreate`**: HTTP Request nodes strip input binary from their output. A `Merge Upload Data` Code node between `stagedUploadsCreate` and the PUT re-attaches the processed binary via `$('Build stagedUploadsCreate Vars').first().binary`.
- **`Get Shopify Token` gates image chain**: `Get Shopify Token` → `Download Image` (explicit connection) ensures the token is available before `stagedUploadsCreate` runs.
- **`patchNodeField` find/replace mangles `$` in the replacement string** (`$'`, `$&`, `$1`… are JS replacement tokens) — corrupted a regex end-anchor into a Postgres query. For any value containing `$`, use `updateNode` with COMPLETE parameters (+ credentials), which stores the string verbatim.
- **Postgres `executeQuery` `query` field resolves inline `{{ }}` natively without a `=` prefix** — the n8n-mcp "Mixed literal … requires = prefix" error is a **false positive** for SQL-editor fields; match the existing live-verified sibling nodes (no `=`).
- `productUpdate(input:…)` is deprecated in 2025-07 → use `productUpdate(product: ProductUpdateInput)`. Schema-validated via shopify-mcp.
- **Shopify product/variant/inventory ids overflow `integer`** — `tl_product_optimizations.wc_product_id` had to become `bigint` (5 dependent views dropped + recreated). Any id column storing a Shopify id must be bigint/text.
- After any partial update on a scheduled workflow, deactivate→reactivate to re-register the Schedule Trigger. Webhooks re-register on save-while-active.
- Slack v2.4 block-message nodes throw a harmless validator `operation` error — pre-existing in the frozen sources; ignore.

---

## Go-live — ✅ Complete (2026-06-06)

Publisher activated 2026-06-06. Detector enqueuing from the 771-product onboarding view; Publisher drafting every 5 min and posting to `#product-publisher` with Publish button. Pipeline live.

## Out of scope / follow-ups

- **AI title hallucination** (optimiser invented a model number from a Qdrant example) — prompt-tune in Phase 11 quality pass.
- The Handler's `map_to_existing` (Phase 11 attribute-mapping) branch has a pre-existing malformed Switch connection — dead button, left for Phase 11.
- Attribute/term/mapping HITL branches still target WooCommerce — Phase 11.
- ~5 verification test products left as drafts on the store — delete in admin.
