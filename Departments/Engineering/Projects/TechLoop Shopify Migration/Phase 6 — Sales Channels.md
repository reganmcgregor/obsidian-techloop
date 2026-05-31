---
status: done
notion_page_id: 3568c3d3-c03e-8123-a2ec-ccce0ddd2fbc
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-25T00:00:00Z
---

# Phase 6 — Sales Channels

## Context

Wire up the free first-party Shopify sales channels: Google & YouTube (Google Merchant Center + Shopping feed), Shop App, Facebook & Instagram. Unblocked by [[Phase 8 — Cutover]] (DNS flip 2026-05-24) — domain verification required the canonical `www.techloop.com.au`.

## Outcome — Done (with Facebook/Pinterest deferred)

### Live

- ✅ **Google & YouTube channel** — connected. Google Merchant Center configured for `www.techloop.com.au`. Shopify Standard Product Taxonomy maps natively to Google Product Categories (no manual GPC mapping required). Shopping feed live.
- ✅ **Shop channel (Shop App)** — set up. Storefront discoverable in the Shop app for AU customers.
- ✅ **GSC verification + sitemap submission** — `www.techloop.com.au` added to Google Search Console; auto-generated sitemap (`https://www.techloop.com.au/sitemap.xml`) submitted. Re-indexing on the new canonical domain underway.
- ✅ **GA/GTM** — analytics tracking installed on the live storefront.

### Deferred

- ⏸️ **Facebook & Instagram channel** — deferred. Reason: products and collections need attribute enrichment before catalog sync is worthwhile (per user, *"we really need to fix up the products and collections first"*). Connects directly to the [[Attribute Strategy (deferred)|Phase 11 attribute rebuild]] — FB/IG catalog quality depends on rich, well-classified product data.
- ⏸️ **Pinterest** — same reason as FB/IG, deferred until catalog data quality improves.

## Open / next

- Once Phase 11 (Attribute Rebuild) lands and the catalog is fuller, revisit FB/IG + Pinterest channels — the product feed will be materially richer and the channels worth the setup effort.
- Watch GMC for product disapprovals in the first weeks post-flip (common issues: missing GTIN, category mismatches, image policy). Address as they surface.
