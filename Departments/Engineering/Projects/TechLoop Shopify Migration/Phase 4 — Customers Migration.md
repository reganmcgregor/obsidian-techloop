---
status: done
notion_page_id: 3568c3d3-c03e-8186-9bc2-d882077ef9c8
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-22T14:53:03Z
---

# Phase 4 — Customers Migration

## Context

Move the WooCommerce customer base to Shopify. Per the [[Phase 0 — Setup & Decisions]] locked decision, **customer accounts only** migrate — order history stays in WooCommerce (no value in porting it; WC remains readable until decommission in Phase 13). Independent of catalog migration; ran in parallel with [[Phase 2 — Catalog Data Migration]].

## Outcome — Done

- WC customers exported, transformed to Shopify customer-CSV format, and imported.
- **Live Shopify customer count: 11** (verified via Admin API `customersCount`, 2026-05-23). The WC customer base was small — expected for a side-hustle store where most traffic converted as guest checkout.
- Order history deliberately **not** migrated (stays in WC per Phase 0 decision).

## Notes

- Account-activation / password-reset invitations are best sent **after** Phase 8 cutover (once the storefront is live and un-gated) so customers land on the real store, not the password page. Confirm send before go-live.
- Marketing-consent state carried across from WC where present; verify opt-in flags before any post-launch email send (ties into Marketing email/newsletter workflows).
