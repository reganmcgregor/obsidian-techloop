---
status: done
notion_page_id: 3568c3d3-c03e-813f-9640-e42a26a1ea53
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-22T14:53:03Z
---

# Phase 3 — Pages, Redirects, SEO Plumbing

## Context

Preserve SEO equity across the platform change: migrate static informational pages, build 301 redirects from every live WooCommerce URL to its Shopify equivalent, and lay the SEO plumbing (collection taxonomy metafield, brand pages). The URL structure changes (WC `/product/{slug}` → Shopify `/products/{handle}`; WC nested category paths → flat `/collections/{handle}`), so redirects are critical to avoid 404s and ranking loss. Runs in parallel with [[Phase 2 — Catalog Data Migration]].

## Outcome — Done (with minor remaining items)

### Redirects — 1,512 live (verified 2026-05-23)

A full audit on 2026-05-23 compared the live WooCommerce RankMath sitemap (~1,299 URLs: 1,210 products, 57 categories, 15 tags, 16 pages, 1 local) against the Shopify redirect set. **Coverage was 99.5%** — only 7 gaps, of which 4 were created and 3 intentionally skipped.

| WC URL type | Sitemap URLs | Already covered | Action |
|---|---|---|---|
| Products (`/product/…`) | 1,210 | 1,209 | +1 created |
| Categories (nested paths) | 57 | 56 | +1 created |
| Tags (`/tag/…`) | 15 | 14 | +1 created |
| Pages | 16 | 11 + 1 native (home) | +1 created (`/about`), 3 skipped |

**Created 2026-05-23** (via `urlRedirectCreate`):
- `/product/amd-ryzen-3-4100-…-wraith-stealth` → `/products/…` (handle match)
- `/computers/components/computer-cases` → `/collections/computer-cases`
- `/tag/black-keyboards` → `/collections/black-keyboards`
- `/about` → `/pages/about`

**Skipped (no Shopify equivalent, user decision 2026-05-23):** `/wishlist`, `/compare` (WC theme features, no native Basic-plan page), `/ecommerce-creative-home-page` (leftover WC demo homepage). Left to 404.

Redirect count moved **1,508 → 1,512**. Brand redirects (124 × `/brand/{slug}` → `/collections/{slug}`) were already in place from the brand-collections backfill.

> Audit working files: `_scratch/redirect-audit/` (`diff.mjs`, `create.mjs`, `diff_results.json`, `create_results.json`).

### Pages migrated

5 Shopify pages live: **about, contact, help, warranty, shipping-information**. WC informational pages (`/returns-and-warranty`, `/shipping`, `/help`, etc.) resolve via redirects to these or to Shopify policy pages. Pages were rebuilt to fit Shopify's structure rather than 1:1 replicated (no blogs to migrate — none existed).

### SEO plumbing

- **`custom.collection_type` metafield** defined and backfilled across **all 463 collections** (8 enums: category, brand, series, attribute, tag, brand+category, brand+category+attribute, special). Pinned, single-line text.
- Standard Product Taxonomy on products maps natively to Google Product Categories — Shopping-feed continuity preserved (feed channel wiring is Phase 6).

## Remaining items

- **Meta titles / descriptions QA** — confirm Rank Math meta titles/descriptions carried across to Shopify SEO fields per product/collection (not yet spot-audited at scale).
- **3 skipped orphan pages** — `/wishlist`, `/compare`, `/ecommerce-creative-home-page` will 404 by design; revisit only if the Hyper theme exposes native wishlist/compare URLs worth pointing at.
- **robots.txt / canonical** — Shopify manages these natively; sanity-check before Phase 8 cutover.
