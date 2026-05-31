---
status: done
notion_page_id: 3568c3d3-c03e-8198-ab10-c55b2ce304a5
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-24T13:15:00Z
---

# Phase 8 — Cutover

## Context

DNS flip from WooCommerce to Shopify. Brainstormed and **executed in the same session** on 2026-05-24, ~2 hours end-to-end. Approach: silent flip, no customer pre-announce, with WC kept running as the rollback safety net (Phase 13 decom comes later). Per the meta-plan: "Shadow build → hard DNS flip."

## Locked decisions (brainstormed 2026-05-24)

| | Decision |
|---|---|
| Primary domain | `www.techloop.com.au` (matches existing WC canonical — preserves SEO equity) |
| DNS provider | Cloudflare (delegated from GoDaddy) |
| CF proxy | **Grey-cloud** (DNS-only) — Shopify already uses CF anycast for storefront edge; orange-cloud on Shopify's CF IPs causes proxy loops (524 errors) |
| Order numbering | Shopify default starting (#1001 onwards) — continuous from WC #24081 not natively possible on Basic without a paid app. Prefix set to `TL-` so orders display as `#TL-1001`, `#TL-1002` etc — visually distinct from the WC era. |
| Customer notification | Silent flip (no email blast, no pre-announce) |
| WC post-flip | Keep running ≥7 days as rollback safety. Decommission is Phase 13. |
| DNS edit ownership | User edited Cloudflare manually (Shopify domain mutations not exposed via Admin API) |

## What was done

1. **Domains added in Shopify admin UI** (Settings → Domains → Connect existing). Both `techloop.com.au` and `www.techloop.com.au` connected. Shopify's API does not expose `domainCreate` — admin UI only.
2. **DNS records updated in Cloudflare** to the records Shopify provided:
   - `A @ → 104.21.36.129` (CF anycast, grey-cloud)
   - `A @ → 172.67.194.95` (CF anycast, grey-cloud)
   - `AAAA @ → 2606:4700:3035::6815:2481` (grey-cloud)
   - `AAAA @ → 2606:4700:3030::ac43:c25f` (grey-cloud)
   - `CNAME www → shops.myshopify.com` (grey-cloud)
   - **Note:** Shopify now uses Cloudflare anycast IPs for storefront edge (migrated from the legacy `23.227.38.65` Shopify apex IP). The CF IPs Shopify hands out are *theirs*, not the merchant's CF account.
3. **Primary domain set** to `www.techloop.com.au` in Shopify admin. Triggers:
   - Canonical URLs on storefront regenerate to `www.techloop.com.au`
   - `.myshopify.com` storefronts auto-301 redirect to `www.techloop.com.au` (resolves the pre-flip canonical-pollution risk)
4. **Old CF proxy removed** on apex DNS records (grey-cloud applied) to stop the orange-cloud-on-Shopify's-CF-IPs proxy loop that was returning 524 errors.

## Verification (post-flip)

- ✅ `www.techloop.com.au` returns HTTP 200 with valid Shopify-issued Let's Encrypt SSL cert (issued 2026-05-24 12:15:34Z).
- ✅ `https://techloop-7.myshopify.com/` returns HTTP 301 → `https://www.techloop.com.au/`.
- ✅ Canonical on www homepage = `https://www.techloop.com.au/`.
- ✅ Existing 1,512 redirects continue to resolve at the new origin (sample-checked).
- ⚠️ **Apex `techloop.com.au` initially returned HTTP 524 (CF timeout loop), then 403** while serving an old CF-issued cert (Google Trust Services, Apr 1) instead of the new Shopify cert. Resolution path: wait for Shopify Let's Encrypt provisioning on the apex hostname (5–30 min after DNS settles), or click "Reissue SSL certificate" on the apex row in Settings → Domains. **User self-resolving.**

## Open items (post-flip, async)

- **Apex SSL cert provisioning** — see above. User will resolve. Once done, bare `techloop.com.au` → 301 → `www.techloop.com.au`.
- **Shopify Payments approval** ([[Phase 7 — Shipping & Payments]]) — Stripe is currently the live gateway (existing config, still works); switch to Shopify Payments when approval clears (2–5 business days).
- **Afterpay merchant agreement** ([[Phase 7 — Shipping & Payments]]) — apply, enable as alt payment method post-approval.
- **Hyper shipping-page template** — handled in the theme session. Customer-facing page won't match the API-pushed body until the template is updated.
- **GA/GTM migration** — Shopify analytics is on by default. Install GA4 + GTM at the new domain when ready (no immediate traffic loss; can happen any time post-flip).
- **GSC site verification** — add `www.techloop.com.au` as a property in Google Search Console; submit the auto-generated `https://www.techloop.com.au/sitemap.xml`.
- ~~**Order numbering**~~ — RESOLVED 2026-05-25. Prefix set to `TL-` in Settings → General → Order ID. First Shopify order displays as `#TL-1001`.

## Rollback (not used)

If anything had gone badly wrong, the documented rollback was: revert CF DNS to the previous CF anycast IPs (`104.21.36.129` / `172.67.194.95` were the previous values too, but with orange-cloud proxy to WC origin). TTL=300 meant a 5-minute reversion window. **Not invoked.**

## Next phase

[[TechLoop Shopify Migration|Phase 6 (Sales Channels)]] is now unblocked — Google & YouTube and Facebook/Instagram channels can verify the canonical domain `www.techloop.com.au`. Then [[Attribute Strategy (deferred)|Phase 11]] post-launch attribute rebuild.
