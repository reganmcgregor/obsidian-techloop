---
status: deferred
notion_page_id: null
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-22T15:30:00Z
---

# Attribute Strategy (deferred — to be picked up in Phase 11)

## Status

**Brainstormed but not specced.** User paused brainstorming 2026-05-24 to push ahead with current phases. Decisions below are agreed; the full design doc + implementation plan was not written. Pick this back up at the start of Phase 11.

## Context

Today's Shopify state (verified 2026-05-23):
- 95 `custom.*` metafield definitions populated from the WC migration.
- Only **4 `shopify.*` definitions exist** (`color-pattern`, `material`, `memory-technology`, `memory-form-factor`) — Phase 1 planned 15 keep-standard handles; the others were never auto-created because no product wrote a value for them.
- Per-product attribute coverage is patchy: RAM products well-covered, other categories thin (e.g. Computer Cases missing `chassis-type`, `side-panel-style`, `color`, `compatible-motherboard-form-factor`).
- This makes ~20 smart collections degrade to category-only rules (RGB Memory, Black Keyboards, Mechanical Keyboards, etc.) and means Hyper theme filters under-populate.

## Decisions locked in brainstorm

1. **Launch posture:** Go live (Phase 8 cutover) with current attribute state. Don't block cutover on attribute completeness. Ship the rebuild as one decisive post-launch project rather than piecemeal interim backfills.

2. **Source for the rebuild values:** Reuse the already-extracted, already-HITL-approved attribute values that live in Supabase mirror tables (`tl_wc_products_mirror` + enrichment tables). No fresh scrape pass. The values are the same as WC has — Supabase is operationally cleaner because it persists past Phase 13 (WC decommission).

3. **Architecture — one push path:** Build `TL_Attribute_Pusher_Shopify` as a single n8n workflow that takes `{wc_id}` as input, reads Supabase, maps to Shopify metafields via the existing `taxonomy.ts` mapper + `SHOPIFY_KEY_OVERRIDES`, writes via `metafieldsSet`. Used **both** by the bulk backfill (Phase 11.A — script fires it against all 1,209 products) **and** by the ongoing enrichment chain (Phase 11.B — HITL "approve" button triggers the same workflow). Avoids two parallel codebases.

4. **No HITL on the bulk push:** Values were previously HITL-reviewed before they landed in WC. Re-reviewing is overhead. Bulk push runs unattended.

5. **Shopify Magic AI suggestions:** Out of scope. Existing TechLoop policy stands (defer; manual accept per product during pre-launch QA if needed).

6. **Approach:** A + audit — bulk push, then an automated gap-analysis Google Sheet listing products with thin `shopify.*` coverage per category. The Sheet drives a later optional targeted scrape pass (Phase 11.C) if needed.

## Phase 11.0 — Attribute Mapping Audit (the prerequisite we discussed but didn't finalise)

Discrete desk-based exercise that **can run in parallel with Phases 6/7/8** (no dependency on cutover). Goal: update the WC→Shopify handle map so the pusher knows where each value should go.

- **Inputs:** the 95 live `custom.*` defs (with top-5 sample values per attr), the canonical Shopify product-taxonomy data (3,310 attributes), category context (which categories use each attr).
- **Auto-propose:** for each `custom.*`, propose 0–3 candidate `shopify.*` matches via fuzzy matcher + LLM semantic pass. Output to a Google Sheet.
- **HITL review (in the Sheet):** mark each row as **promote** (move to shopify.*), **mirror during transition** (write both), **keep-custom** (no Shopify equivalent — `pcie-slots`, `mtbf`, `fan-airflow-cfm` etc), or **deprecate**.
- **Apply phase:** generate expanded `SHOPIFY_KEY_OVERRIDES`, DDL to create missing `shopify.*` definitions, deprecation list for unused `custom.*` defs.
- **Category-level pass:** for each Shopify product category in use, compare our `custom.*` attrs against Shopify Magic's category-suggested attributes — promote exact matches.
- **Effort:** ~1h review work in Sheet + ~3h scripting + ~half-day DDL+overrides apply.

## Phase 11.A — Bulk backfill (after 11.0 + after cutover)

Once mapping is locked and `TL_Attribute_Pusher_Shopify` exists, a `techloop-migration` script paginates `tl_wc_products_mirror` and POSTs each `wc_id` to the n8n workflow webhook. Resume-on-failure, progress log, idempotent. Hours-long unattended run.

After completion, an automated audit script writes a Google Sheet listing products with thin `shopify.*` coverage per category — drives the optional Phase 11.C scrape-for-gaps decision.

## Out of scope (separate decisions later)

- **Phase 11.B — ongoing enrichment rebuild:** re-pointing scrape→LLM→HITL chain at the new pusher. Different design conversation, but builds naturally on 11.A.
- **Phase 11.C — targeted scrape for gaps:** only if the audit Sheet shows meaningful gaps worth filling.
- **Phase 14 — Shopify-native AI integrations:** Shopify Magic acceptance automation, AI taxonomy suggestions — long-tail enhancement, not the rebuild itself.

## Picking this up later

Re-enter brainstorm at Section 4 (data flow / per-product mapping detail) and Section 5 (smart collection refresh + audit Sheet schema). Sections 1–3 are settled and captured above.
