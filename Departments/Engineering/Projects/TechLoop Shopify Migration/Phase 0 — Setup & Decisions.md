---
status: done
notion_page_id: 3568c3d3-c03e-8196-b96a-e9420431e619
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-22T14:53:03Z
---

# Phase 0 — Setup & Decisions

## Context

Establish the foundations for the WC→Shopify migration before any catalog work: provision the Shopify store, stand up the repos, enable Shopify's Standard Product Taxonomy, freeze the n8n automation, archive the abandoned Ecomus theme project, and lock the irreversible architectural decisions. See [[TechLoop Shopify Migration]] for the full meta-plan and rationale.

## Outcome — Done

### Store & plan
- **Shopify Basic** plan (~$56 AUD/mo, $42 annual), currency AUD. No paid apps — build-don't-buy default.
- **Permanent myshopify domain:** `zc30tg-fi.myshopify.com` — used for CLI auth and all API integrations (the vanity alias `techloop-7.myshopify.com` errors out OAuth callbacks).
- **5 Locations** configured (NSW, QLD, SA, VIC, WA); NSW is the only stocked location (single-location inventory pattern — Leader feed only carries aggregate stock).
- **Standard Product Taxonomy** enabled (maps natively to Google Product Categories for the Shopping feed).

### Repos
- **`techloop-shopify`** (long-lived) — theme at `theme/`, custom app, MCP integrations.
- **`techloop-migration`** (throwaway) — one-off WC→Shopify import scripts.
- Vault stays strategy/knowledge only; code never lives here.

### Supabase mirror
- `tl_*` mirror tables stood up for the transition period so most n8n workflows talk to Supabase, not WC/Shopify directly — isolating the platform-specific layer to ~5 workflows (see [[Shopify Migration Considerations]]).

### Theme
- **FoxEcom Hyper v1.3.3** purchased 2026-05-16 (themeStoreId 3247). The earlier clean-room theme build (locked 2026-05-12) was abandoned same day — 18 sub-tickets archived, replaced by the Hyper sub-task tree 5.A–5.I. See [[Phase 5 — Storefront Theme (Hyper)]].
- Previous WC theme work (Ecomus) killed and archived — was WP-specific.

## Locked decisions

| Decision | Resolution |
|---|---|
| Customers vs order history | Migrate **customer accounts only** (CSV); order history stays in WC. See [[Phase 4 — Customers Migration]]. |
| Variants vs individual products | **Individual products** (one SKU = one product) — better SEO/search, simpler automation. |
| Linked/sibling products | Deferred to **Phase 5b** (post-launch) via Metaobject "Product Group" pattern. |
| Brands | `vendor` field + **Brand metaobject** (self-referencing for sub-brands). |
| Attributes | Shopify **metafields** (no WC global-attribute equivalent); filtering via free Search & Discovery app. |
| Categories | Flat collections + Standard Product Taxonomy; visual hierarchy via nav menus, not deep collection trees. |
| Cutover model | Shadow build behind password gate → hard DNS flip (Phase 8). |
| Image source | WC gallery only — **never** the Leader supplier feed (raw/unedited). |

## Why

Escape the self-hosted WooCommerce infrastructure burden (bot mitigation, Cloudflare rules, server overhead) so effort goes into building the store, not patching infra — and move toward mobile-first, Slack-driven management on a platform with native AI/MCP direction.
