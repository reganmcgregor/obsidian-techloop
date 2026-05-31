---
status: done
notion_page_id: 3568c3d3-c03e-81fb-ab53-e7ef561c9437
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-24T12:30:00Z
---

# Phase 7 — Shipping & Payments

## Context

Configure shipping rates/zones + a live payment processor so customers can check out on Shopify, unblocking the [[TechLoop Shopify Migration|Phase 8 cutover]] (DNS flip imminent within days as of 2026-05-24). This phase resolves the meta-plan's open decision (Shopify Payments vs Stripe) and replaces the WC freight model with a simpler flat-Standard + scaled-Express structure. The customer-facing shipping page is updated to match.

## Locked decisions (brainstormed 2026-05-24)

### Payments stack

| Method | How | Rationale |
|---|---|---|
| **Shopify Payments** (primary) | Settings → Payments → Activate (admin UI) | Removes the 0.5–2% third-party-gateway fee Basic charges on Stripe. Unlocks Shop Pay, native Apple/Google Pay, Shop Pay Installments. Requires AU business verification — 2–5 business days. |
| **Shop Pay Installments** | Native with Shopify Payments | Built-in BNPL, no extra setup or fee, 4 interest-free instalments. |
| **Apple Pay + Google Pay** | Native with Shopify Payments | Auto-enabled. |
| **Afterpay** | Separate Afterpay AU merchant agreement → enable as Shopify alt method | Dominant AU BNPL for electronics ($1k–$2k AOV sweet spot). ~6% merchant fee. |
| **Bank Transfer** | Shopify Manual Payment Method | BACS-equivalent. Order stays "Pending" until you mark Paid manually. For occasional B2B / large-order customers. |

**Removed/not used:**
- Existing Stripe gateway disabled once Shopify Payments is approved (don't run both — confuses customers and pays the 2% Shopify-on-Stripe fee).
- No PayPal (never offered on WC, no demand signal).
- No Zip (Afterpay covers the BNPL bucket).

### Shipping rates (already configured live, verified 2026-05-24)

**Domestic AU only.** International zone removed.

| Method | Rate | Conditions |
|---|---|---|
| **Standard** | **$20 flat** | Australia-wide |
| **Standard (free)** | $0 | TOTAL_PRICE ≥ $300 AUD |
| **Express ≤1kg** | $15 | TOTAL_WEIGHT 0–1kg |
| **Express 1–3kg** | $20 | TOTAL_WEIGHT 1.0001–3kg |
| **Express 3–5kg** | $25 | TOTAL_WEIGHT 3.0001–5kg |
| **Express 5–10kg** | $40 | TOTAL_WEIGHT 5.0001–10kg |
| **Express 10–20kg** | $80 | TOTAL_WEIGHT 10.0001–20kg |

The Express scale mirrors Leader's wholesale air-freight pricing (1/3/5/10/20kg bands at $12/$15/$17.50/$40/$80 ex-GST). TechLoop's retail Express rates mark up the lower bands ($15/$20/$25 vs $12/$15/$17.50) and pass through the higher bands at cost.

### Why flat Standard, not postcode zones

Brainstormed and rejected: replicating WC's NSW postcode zones at nationwide scale via Auspost's "Capital Cities / Major Areas / Rural" classification. Postcode-level rate filtering isn't a first-class Shopify feature — it requires either:
- A custom carrier-service webhook (Carrier Service API, needs Basic-annual or Grow plan, plus n8n infra), or
- A third-party shipping app (Starshipit/Shippit, $30–$80/mo recurring).

Decision: ship with **flat $20 Standard** as the launch model. Simpler customer message ("$20 flat, free over $300"), no infrastructure overhead, no plan upgrade. Revisit postcode-based pricing post-launch if margin analysis shows rural orders are eroding contribution materially.

### Customer-facing shipping page

Published 2026-05-24 to `/pages/shipping` ([live URL](https://techloop-7.myshopify.com/pages/shipping)). Content drafted in TechLoop's [[Tone of Voice|Brutally Transparent / Aussie pragmatic]] voice; covers rates, dispatch cutoff (3:30pm AEST same-day), delivery time bands (1–3 / 2–5 / 5–10 business days for Standard, next-day / 2–4 days for Express), PO Box handling (Express only), bulky goods, genuinely remote postcodes, free-shipping fine print, tracking + shipping partners. Links to [[Phase 7 — Shipping & Payments|Warranty & Returns]] for the returns flow.

## Setup status (as of 2026-05-25)

1. ✅ **Shopify Payments activated** — approved and live as primary processor.
2. ✅ **Stripe gateway disabled** (replaced by Shopify Payments).
3. ⏳ **Afterpay merchant application — under review** (a few business days; not blocking the store).
4. ✅ **Shop Pay Installments** — auto-enabled with Shopify Payments.
5. ✅ **Bank Transfer Manual Payment Method** — configured (or N/A if not enabled).
## Open items (async)

- ⏳ **Afterpay** — application under review, will activate as Shopify alt method on approval.
- **Tax / GST** — verify AU GST at 10% is configured under Settings → Taxes & duties (small spot-check, not blocking).
- **Hyper shipping page template** — handled in the theme session.

## Out of scope (later phases)

- **Phase 6 — Sales Channels** (Google/YouTube + Facebook/Instagram): blocked on canonical domain verification (post DNS-flip).
- **Phase 8 — Cutover** (DNS flip): starts when Shopify Payments approved + remaining setup #1–#7 above are done.
- **Postcode-based shipping** (deferred): revisit post-launch if flat-rate margin proves unsustainable on rural orders.

## Verification

- Live page: `https://techloop-7.myshopify.com/pages/shipping` renders the new content.
- Delivery profile via Admin GraphQL shows 1 Standard $20 + 1 conditional $0 (≥$300) + 5 Express bands.
- A test cart at $50 shows: Standard $20 + Express $15 (or weight-appropriate Express). A test cart at $350 shows: Standard FREE + Express options.
- Payments settings show Shopify Payments + Afterpay + Shop Pay + Bank Transfer as available methods (after each is enabled).
