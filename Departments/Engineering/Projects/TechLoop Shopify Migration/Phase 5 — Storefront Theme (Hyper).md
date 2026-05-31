---
status: in-progress
notion_page_id: 3628c3d3-c03e-81a1-a365-c6a69948ac97
notion_parent_id: 405da260-be63-4c37-b81e-ac4fe5512021
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-19T15:49:26Z
---

# Phase 5 — Storefront Theme (Hyper)

## Overview

Set up the **FoxEcom Hyper** theme on `zc30tg-fi.myshopify.com` (alias `techloop-7.myshopify.com`) so the storefront is ready to publish at cutover (Phase 8).

**Approach changed 2026-05-16:** abandoned the clean-room reimplementation (too long). Now: install Hyper, pick a demo, rebrand with the TechLoop Claude design system, wire up the imported catalog, configure all customer-visible surfaces, ship.

## Strategic decisions (locked 2026-05-16)

| Decision | Choice |
|---|---|
| Theme | FoxEcom Hyper v1.3.3 (purchased via Shopify Theme Store, themeStoreId 3247) |
| Build approach | Import a Hyper demo + rebrand (no blank scaffold) |
| Build location | Live store directly + Shopify CLI local dev to `~/Code/techloop-shopify/theme/` |
| Brand | Claude design system — brand is locked |
| Wishlist + Compare + Quick View | Folded into Phase 5 (Hyper toggles, no separate apps) |
| Linked Products (custom) | Stays in Phase 5b (post-launch) |
| Phase 3 (pages, redirects, SEO) | Asserted Done — but Phase 3 Notion task = Not Started. **Reconcile.** |
| Cutover | Theme set as Active but password-gated until Phase 8 DNS flip |

## Critical files & paths

- **Live store:** `zc30tg-fi.myshopify.com` (permanent), `techloop-7.myshopify.com` (primary alias)
- **Local repo:** `~/Code/techloop-shopify/theme/`
- **Branch:** `theme/hyper-base` → tag `theme/hyper-v1.0` at activation
- **Hyper docs:** https://docs.foxecom.com/hyper-theme
- **Claude design system:** https://api.anthropic.com/v1/design/h/D6C4vinwsadVd9ViZtY3uQ — read README, implement `ui_kits/store/index.html`
- **Catalog state:** 1,209 ACTIVE products + Phase 1 metafield definitions + Brand metaobject (52 entries) already live; see [[Phase 2 — Catalog Data Migration]]

## Live-verified status (2026-05-20)

Notion has every sub-task as "Not Started". Live store check via Shopify CLI shows otherwise:

- **5.A — Install Hyper + import demo + CLI wire-up:** **Effectively Done.** Hyper v1.3.3 is installed, renamed to "TechLoop", and set as MAIN (Live) theme on the store. `config/settings_schema.json` confirms `theme_name: "Hyper"`, `theme_author: "FoxEcom"`. Local repo + branch + CLI wiring needs separate verification.
- **5.B–5.H:** Not Started (best inference — theme settings still on FoxEcom defaults, no Claude brand tokens visible). Sub-tasks 5.C (catalog wire-up), 5.D (homepage), 5.E (navigation), 5.F (core templates), 5.G (secondary templates), 5.H (QA) require Liquid / theme-editor inspection to confirm.
- **5.I — Activate (password-gated):** **Likely Done** — theme is MAIN. Password-gate state (Online Store → Preferences) not verifiable via Admin GraphQL; needs admin UI check.

## Sub-task tree

| # | Task | Depends on |
|---|---|---|
| 5.A | Install Hyper + import demo + CLI wire-up | — |
| 5.B | Apply brand + global theme config | 5.A |
| 5.C | Catalog wire-up (metafields, Brand metaobject, S&D filters) | 5.A (parallel with 5.B) |
| 5.D | Homepage | 5.B |
| 5.E | Navigation | 5.B (parallel with 5.D) |
| 5.F | Core templates (PDP, Collection, Search, Cart) | 5.B + 5.C |
| 5.G | Secondary templates (Account, 404, password, static page templates) | 5.B + 5.F |
| 5.H | QA pass (responsive, Lighthouse, cross-browser, sample products) | 5.B–5.G |
| 5.I | Activate (password-gated) | 5.H |

### 5.A — Install Hyper + import demo + CLI wire-up

Upload Hyper theme zip to the store, pick the closest demo (likely electronics/tools), import demo content, pull to local repo via Shopify CLI.

- **Steps:** Upload Hyper zip → review demos → import → `shopify theme pull --theme=<draft-id>` into `theme/` → initial commit on `theme/hyper-base` → confirm `shopify theme dev` serves locally without console errors.
- **Done when:** Hyper exists on the store with demo content imported; local repo matches remote; `theme/hyper-base` initialised; `shopify theme dev` runs cleanly.
- **Live status:** Hyper IS live on the store (renamed "TechLoop"). Local repo + branch state not verified from this session.

### 5.B — Apply brand + global theme config

Import the TechLoop Claude design system into Hyper's global theme settings; enable Wishlist/Compare/Quick View; install any free third-party app blocks.

- **Inputs:** design system at the Anthropic URL above; implement `ui_kits/store/index.html`.
- **Steps:** Logo + favicon (SVG + PNG fallbacks) uploaded to Files → map design tokens to Hyper theme settings (colour schemes, typography, buttons, cards, borders, motion/transitions) → enable Hyper features Wishlist/Compare/Quick View → install free Shopify app blocks (Reviews, Shop Pay messaging) → smoke check brand visible on header, footer, button across one PDP and one collection.
- **Done when:** theme settings reflect design tokens; logo + favicon set; Wishlist/Compare/Quick View enabled; app blocks installed; brand visible on key surfaces.

### 5.C — Catalog wire-up (metafields, Brand metaobject, S&D filters)

Configure Hyper's PDP and Collection templates to render Phase 1 metafield definitions and the Brand metaobject. Configure Search & Discovery filters to match Phase 1 attributes.

- **Inputs:** Phase 1 metafield definitions (live), Brand metaobject (52 entries, live), 1,209 imported products.
- **Steps:** PDP renders metafield specs table → PDP renders Brand metaobject block → Brand pages template wired → S&D filters per Phase 1 metafield definitions → spot-check 10–20 products spanning categories.
- **Done when:** PDP renders Phase 1 metafields; Brand metaobject on PDP + brand pages; S&D filters operational on collections; spot-check passes.

### 5.D — Homepage

Replace Hyper demo homepage content with TechLoop content blocks; mobile-first.

- **Sections:** Hero (announcement + CTA) → Featured collections → USP / trust (free shipping, NSW dispatch, returns) → Brand showcase (Brand metaobject grid) → Featured products / new arrivals → Blog preview → Footer CTAs (newsletter signup).
- **Done when:** all demo content replaced; renders correctly mobile-first across 375 / 768 / 1280 / 1600 breakpoints; no demo placeholder images or lorem text.

### 5.E — Navigation

Configure main menu (mega menu if Hyper supports), footer menus, mobile menu.

- **Steps:** main menu structure → mega menu with category + featured brands → footer menus (Help, About, Legal, Connect per Phase 3 page structure) → mobile menu parity.
- **Done when:** main menu desktop + mobile; mega menu (if used) populated; footer links all Phase 3 static pages; mobile menu functional.

### 5.F — Core templates (PDP, Collection, Search, Cart)

Pick + customise Hyper template variants for the four highest-traffic surfaces.

- **PDP:** Hyper spec-heavy electronics variant; sections gallery / specs / Brand block / related / recently viewed.
- **Collection:** Hyper variant with filter UI, sort, product card density.
- **Search:** predictive search + search results page.
- **Cart:** drawer with free-shipping goal, upsell zone, sticky ATC variants.
- **Done when:** all four surfaces configured; section blocks tuned; no console errors; mobile + desktop balanced.

### 5.G — Secondary templates

Configure remaining customer-visible templates.

- **Templates:** customer account (decide classic vs new accounts; login / register / order history / account), 404 (branded with helpful CTAs), password (branded coming-soon — used pre-cutover), static page templates (About / Contact / FAQ / Returns / Shipping / Privacy / T&Cs render Phase 3 page content).
- **Done when:** all templates configured; static page content renders without layout breakage; customer account flow works end-to-end.

### 5.H — QA pass

Cross-device, cross-browser, and performance sweep before activation.

- **Responsive:** sweep 9 key surfaces (home, collection, PDP, cart, search, account, 404, password, one static page) across 375 / 768 / 1280 / 1600 breakpoints.
- **Lighthouse:** mobile + desktop on home + PDP + collection. Score must meet or beat the Hyper demo baseline (Performance, A11y, Best Practices, SEO).
- **Cross-browser:** smoke test home + PDP + cart in Chrome, Safari, Firefox.
- **Catalog spot-check:** sample 20–30 products across categories; verify metafields, images, prices, variants render correctly.
- **Console:** zero errors on key surfaces.
- **Done when:** all checks pass; critical blockers fixed before 5.I; cosmetic items deferred to follow-up tickets.

### 5.I — Activate (password-gated)

Switch Hyper from Draft to Active theme; confirm password gate stays in place until cutover (Phase 8).

- **Steps:** switch Hyper to Live → verify password gate still enabled (Online Store → Preferences) → smoke check password-gated URL → tag commit `theme/hyper-v1.0` on `theme/hyper-base`; push tag → update Phase 5 ticket with go-live date.
- **Done when:** Hyper is MAIN; password gate enabled; `theme/hyper-v1.0` tag pushed; all 5.A–5.H sub-tasks Done.
- **Live status:** theme is MAIN — likely already done modulo password-gate verification and the tag push.

## Model & sub-agent strategy

| Sub-task | Recommended runner |
|---|---|
| 5.A | Opus (foundational decisions) |
| 5.B | Sonnet sub-agent (Opus reviews after first surface) |
| 5.C | Sonnet sub-agent (parallel with 5.B) |
| 5.D | Sonnet sub-agent (Opus reviews layout) |
| 5.E | Sonnet sub-agent (parallel with 5.D) |
| 5.F | Sonnet sub-agent (one per surface, fan out) |
| 5.G | Sonnet sub-agent |
| 5.H | Sonnet sub-agent for sweep + Haiku for spot checks; Opus triages |
| 5.I | Opus + person (single decision step) |

**Anti-pattern:** Opus running theme-editor click-throughs or Liquid edits directly. If the orchestrator does more than 3–4 medial edits in a row, spawn a Sonnet sub-agent.

## Definition of Done

1. Hyper installed on the store, set as Active theme, password-gated until cutover.
2. TechLoop brand applied: design system tokens reflected across every surface.
3. All 1,209 imported products render correctly on PDP with Phase 1 metafields visible.
4. Brand metaobject renders on PDP + brand pages.
5. Search & Discovery filters operational and aligned with Phase 1 metafield definitions.
6. Wishlist + Compare + Quick View enabled and functional.
7. Homepage, navigation, cart, search, account, 404, and static page templates all configured and rendering.
8. Lighthouse mobile + desktop ≥ Hyper demo baseline (Performance, A11y, Best Practices, SEO).
9. No console errors on any of the 9 key surfaces.
10. Theme committed + tagged `theme/hyper-v1.0` on `theme/hyper-base` branch.
11. Cross-browser smoke pass (Chrome, Safari, Firefox).

## Implementation note

This phase is planned in this vault but **executed in the `~/Code/techloop-shopify/` repo** in a fresh Claude Code session. This file is the canonical spec; the Notion sub-tasks are the execution checklist. Update sub-task status as each step completes (per [[feedback_notion_progress_tracking]]).
