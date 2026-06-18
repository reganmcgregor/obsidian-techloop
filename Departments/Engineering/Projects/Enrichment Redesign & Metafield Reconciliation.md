---
notion_page_id: null
notion_parent_id: null
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# Enrichment Redesign & Metafield Reconciliation

## Context / why

The WooCommerce decommission ([[Phase 13 — WooCommerce Decommission]]) is complete. The attribute-enrichment → Shopify metafield pipeline now needs a deliberate redesign: WooCommerce attributes and Shopify's standard product taxonomy are not like-for-like, so a direct mapping does not hold. This work was paused and spun off as its own project on 2026-06-18.

See [[TechLoop Shopify Migration]] for the overall migration phase map.

---

## Known issues / backlog

- **`metafieldsSet` atomicity (API 2025-07)** — the mutation appears atomic per product: one invalid metafield blocks the entire product's push. Needs per-metafield error isolation or pre-validation before the push call.

- **`shopify.*` metafields are taxonomy-category-constrained** — pushing a standard metafield to a product in the wrong Shopify category fails with `INVALID_VALUE "owner subtype…"`. Preferred direction: route category-mismatched attributes to a `custom.*` metafield (no category constraint) rather than silently dropping the value.

- **157 `shopify.*` keys are enable-able via `standardMetafieldDefinitionEnable`** — standard definitions are limit-exempt. All 157 are `list.metaobject_reference` and require metaobject vocabulary seeded in `tl_shopify_metaobject_values` plus GID resolution in the Pusher (the deferred TL_Sync_Shopify_Attribute_Vocab work). Three keys are category-scoped with no available template (band / case / dial-pattern).

- **Admin-filterable cap = 50 metafield definitions per product owner type** — must decide which attributes become storefront filters before enabling definitions at scale.

- **Boolean-vs-descriptive modelling** — some `custom.*` definitions are boolean (e.g. webcam) but the extracted data is descriptive ("5MP IR"). Decide per attribute whether it is a yes/no flag or a text/spec field. Tier 1 coercion currently skips non-matching values (no corruption, but those attributes will not populate until remodelled).

- **Orphaned proposals accumulating** — TL_Attribute_Proposer was deleted in Phase 13, but TL_Enrich_Attributes still inserts into `tl_proposed_attributes` and reports "X new proposals" with no consumer (821 rows as of 2026-06-18). Decision needed: silence and stop generating, log quietly, or build a Shopify-native digest/review flow.

- **Cosmetic WC-named renames** — mapping keys `wc_attribute_id`/`wc_slug`, Slack `action_id: wc_publish_product`, and dormant columns (`tl_category_map.wc_category_id`/`wc_category_name`, `tl_product_supplier_status.wc_id`, `tl_sync_log.wc_id`) carry WC-derived names but have no runtime dependency. Optional tidy-up, can be batched with schema work here.

- **Existing read-only audit utilities (INACTIVE):**
  - TL_Audit_Metafield_Definitions (`QdQ3piI6qb9TBGmS`)
  - TL_Audit_Standard_Templates (`Ii9xZdAxO5K475nu`)

---

## Status

Not started — paused/parked pending design.
