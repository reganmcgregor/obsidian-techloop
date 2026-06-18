---
created: 2026-06-18
phase: Phase 13 — WooCommerce Decommission
step: Step 0 — Safety Snapshot
---

# Phase 13 Snapshot Notes

Snapshot taken 2026-06-18 as Step 0 of the WooCommerce decommission.
All operations were read-only. No schema changes, no drops, no deletes.

## Table Row Counts (at time of snapshot)

| Table | Rows | Dump file |
|---|---|---|
| tl_wc_products_mirror | 1,509 | tl_wc_products_mirror.json |
| tl_wc_attributes_mirror | 172 | tl_wc_attributes_mirror.json |
| tl_wc_attribute_terms_mirror | 1,000 | tl_wc_attribute_terms_mirror.json |
| tl_attribute_aliases | 0 | — (empty, no dump needed) |
| tl_shopify_metafield_values_mirror | 0 | — (empty, no dump needed) |

`tl_attribute_aliases` and `tl_shopify_metafield_values_mirror` were empty — no dump files created for these.

## Frozen Workflow Exports

Five workflow JSONs exported to `frozen-workflows/`:

| File | n8n ID | Was active? |
|---|---|---|
| TL_Mirror_WooCommerce_hEUnN85kXJZEG3qZ.json | hEUnN85kXJZEG3qZ | No (FROZEN-2026-05) |
| TL_Inventory_Syncer_WC_ZgqlMhNqh3TJMB6G.json | ZgqlMhNqh3TJMB6G | No (FROZEN-2026-05) |
| TL_Product_Detector_WC_rnUjGGsdD1jdaFIg.json | rnUjGGsdD1jdaFIg | No (FROZEN-2026-05) |
| TL_Slack_Handler_WC_rDloLTYACT56kfpn.json | rDloLTYACT56kfpn | No (FROZEN-2026-05) |
| TL_Attribute_Proposer_hUGA1KFWBTGU8K6T.json | hUGA1KFWBTGU8K6T | **Yes** (active at snapshot time; deactivated in Step 6) |

## Step 1 — Workflows Paused

Three schedule-triggered workflows deactivated (active → false) at 2026-06-18 ~02:32 UTC:

| Workflow | n8n ID | active after deactivation |
|---|---|---|
| TL_Enrich_Attributes | lq8960K0xVwF3Xst | false ✓ |
| TL_Enrichment_Reviewer | OarmHP6DpJvAV0Pb | false ✓ |
| TL_Product_Detector_Shopify | fz50fq1dCmrHhazX | false ✓ |

Verified via n8n_get_workflow (minimal mode) after deactivation — all returned `"active": false`.
