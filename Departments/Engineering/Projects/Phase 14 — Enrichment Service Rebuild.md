---
notion_page_id: null
notion_parent_id: null
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# Phase 14 — Enrichment Service Rebuild

## Context / why a rebuild

[[Phase 13 — WooCommerce Decommission]] is closed. The attribute-enrichment → Shopify metafield pipeline will be rebuilt from the ground up rather than patched. WooCommerce and Shopify attribute models are not like-for-like: the current migrated chain is keyed on `wc_attribute_id`/`wc_slug`, performs a single-namespace push, and has no category awareness — all WC-era assumptions that do not map cleanly to Shopify's standard product taxonomy. A ground-up rebuild is a cleaner path than reconciling these mismatches on top of the existing chain.

**Status:** Not started — parked pending design.

See [[TechLoop Shopify Migration]] for the overall migration phase map.

---

## Design inputs / known issues to architect around

These are requirements and constraints for the new design, carried over from findings made during Phases 11–13.

### metafieldsSet atomicity
`metafieldsSet` (API 2025-07) is atomic per product — one invalid metafield blocks the entire product push. The new design must implement per-metafield isolation (e.g. push one metafield at a time or pre-validate each metafield before the batch call).

### shopify.* category constraints
`shopify.*` standard metafields are taxonomy-category-constrained. Pushing a standard key to a product in the wrong Shopify category fails with `INVALID_VALUE "owner subtype…"`. Rather than dropping mismatched values silently, route them to a `custom.*` metafield namespace (no category constraint).

### Standard metafield definitions — 157 enable-able keys
157 `shopify.*` keys can be enabled via `standardMetafieldDefinitionEnable`; standard definitions are limit-exempt. All 157 are typed `list.metaobject_reference`, which requires:
- metaobject vocabulary seeded in `tl_shopify_metaobject_values`
- GID resolution in the Pusher at write time

Three keys are category-scoped with no available standard template (`band`, `case`, `dial-pattern`) — handle as `custom.*` or omit.

### Admin-filterable cap
The storefront admin-filterable cap is 50 metafield definitions per product owner type. Prioritise filter-relevant attributes (standard-taxonomy filtering matters for discovery) before enabling definitions at scale.

### Boolean-vs-descriptive modelling
Some `custom.*` definitions are modelled as boolean (e.g. webcam) but extracted data is descriptive (e.g. "5MP IR"). Decide per attribute whether to model as a yes/no flag or a text/spec field. Current Tier 1 coercion silently skips non-matching values — no corruption, but affected attributes do not populate until remodelled.

### Orphaned proposals
`TL_Attribute_Proposer` was deleted in Phase 13, but `TL_Enrich_Attributes` still inserts into `tl_proposed_attributes` and reports "X new proposals" with no consumer (821 rows as of 2026-06-18). Decide in the new design: silence the proposals output entirely, log quietly, or build a Shopify-native review/digest flow.

### WC-named identifiers to retire
The rebuild is the natural opportunity to retire these cosmetic WC-named identifiers (no runtime WC dependency, but carry misleading names):

- `tl_attribute_mapping.wc_attribute_id`, `wc_slug`
- `tl_product_attributes.wc_attribute_id`
- `tl_proposed_attributes.wc_attribute_id`
- Slack `action_id: wc_publish_product`
- Dormant unused columns: `tl_category_map.wc_category_id`/`wc_category_name`, `tl_product_supplier_status.wc_id`, `tl_sync_log.wc_id`

### Existing read-only audit utilities (INACTIVE — reusable)
These workflows are inactive and carry no WC assumptions. Reuse or adapt in the new design:

- **TL_Audit_Metafield_Definitions** (`QdQ3piI6qb9TBGmS`)
- **TL_Audit_Standard_Templates** (`Ii9xZdAxO5K475nu`)

---

## Workflows to be superseded

The following n8n workflows from the current enrichment chain will be superseded by the rebuild. Keep active until the replacement is verified.

| Workflow | n8n ID | Role |
|---|---|---|
| TL_Enrich_Attributes | `lq8960K0xVwF3Xst` | Attribute extraction |
| TL_Enrichment_Reviewer | `OarmHP6DpJvAV0Pb` | Human review queue |
| TL_Attribute_Pusher_Shopify | `s06MMF8cRDi0DXhj` | Shopify metafield push |
| TL_Product_Detector_Shopify | `fz50fq1dCmrHhazX` | New product detection |
| TL_Product_Publisher_Shopify | `pgWWBz9f6RXLXEIe` | Shopify product publish |
| TL_Slack_Interaction_Handler_Shopify | `hikIeVV081e76pEv` | Shared Slack handler |

---

## Status

Not started — parked pending design.
