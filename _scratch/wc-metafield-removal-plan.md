# WooCommerce Metafield Removal — STAGED (HOLD)

**Status: HELD — do not execute yet.** Staged 2026-06-26.

137 free-text `custom.*` metafield definitions remain from the WooCommerce migration, now **superseded** by the new structured taxonomy metafields (`shopify.*` Category metafields + our typed `custom.*` fields). Full list: [`wc-metafields-list.csv`](wc-metafields-list.csv) (key, name, type, def GID).

## Why hold until after the theme migration
- The **Hyper PDP currently renders `custom.*` keys** → deleting these would blank the spec tables on most products until the theme renders the new structured fields. Remove **only after** the Hyper theme MVP ships.
- Deleting a metafield **definition deletes all its values** (irreversible). Export values first if any are worth archiving.
- Confirm the **Google Merchant Center** feed does not read these keys.

## Removal method (when greenlit)
Per def, Shopify Admin GraphQL:
`metafieldDefinitionDelete(id: <gid>, deleteAllAssociatedMetafields: true)`
Batch via a small n8n workflow reading the 137 GIDs from `tl_shopify_metafield_definitions_mirror` (namespace='custom', key NOT IN `tl_custom_attribute_registry`). **Re-sync the def mirror first** (it last synced 2026-06-22 → may be stale). Dry-run (list only) → delete in batches with logging.

## Pre-removal checklist
1. [ ] Hyper theme MVP live (renders new `shopify.*` / `custom.*` specs)
2. [ ] GMC feed verified independent of WC keys
3. [ ] (optional) export WC values for archive
4. [ ] Re-sync def mirror, then run the batched delete
