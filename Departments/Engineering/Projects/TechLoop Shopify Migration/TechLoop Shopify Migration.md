---
status: active
notion_page_id: 3558c3d3-c03e-8164-9b2c-ff4243c5147a
notion_parent_id: 7093b771-b742-41cd-9642-49e0b3c6531c
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-24T12:30:36Z
---

# TechLoop Shopify Migration

Migration of techloop.com.au from WooCommerce (self-hosted) to Shopify Basic.

**Related Docs:**
- [[Phase 0 — Setup & Decisions]] — Done — repos, Basic plan, taxonomy, Supabase mirror, Hyper purchase, locked decisions
- [[Phase 1 — Catalog Model]] — Resolved catalog taxonomy mapping (categories, brands, attributes, smart collections)
- [[Phase 2 — Catalog Data Migration]] — Done — 1,209 products imported, schema upgrade, image policy, inventory
- [[Phase 3 — Pages, Redirects, SEO]] — Done — 1,512 redirects (99.5% sitemap coverage), pages, collection_type metafield
- [[Phase 4 — Customers Migration]] — Done — 11 customer accounts imported (no order history)
- [[Phase 5 — Storefront Theme (Hyper)]] — Done — FoxEcom Hyper live; password gate removed 2026-05-24
- [[Phase 6 — Sales Channels]] — Done — Google & YouTube + Shop App live; FB/IG + Pinterest deferred until catalog enrichment
- [[Phase 7 — Shipping & Payments]] — Done — Shopify Payments live (Stripe disabled), shipping config + page live, Afterpay under review
- [[Phase 8 — Cutover]] — Done — DNS flipped 2026-05-24; www.techloop.com.au is primary, .myshopify auto-301s, apex SSL provisioning in flight
- [[Phase 9 — Workflow Rebuild Tier 1 (Inventory)]] — ✅ Done — TL_Mirror_Shopify + TL_Inventory_Syncer_Shopify both live
- [[Phase 10 — Workflow Rebuild Tier 2 (Onboarding)]] — ✅ Done (2026-05-31) — Detector + Publisher + Slack Handler built; category map resolved (426 approved); view yields 771 products; awaiting user go-live (activate Publisher)
- [[Attribute Strategy (deferred)]] — captured brainstorm decisions for Phase 11 rebuild (post-launch); not yet specced
- [[Shopify Linked Products Research]] — Pattern 3 (Metaobject Product Group) — deferred to Phase 5b
- [[Shopify Migration Considerations]] — Pre-decision impact assessment (historical reference only)
- [[System Overview]] — Current n8n automation architecture
- [[Database Schema - TechLoop Inventory]] — Phase 1–4 schema
- [[Database Schema - Product Automation]] — Phase 6 schema

---

## Status (as of 2026-05-23)

### Phase table

| Phase | Title | Status | Notes |
|---|---|---|---|
| 0 | Setup & Decisions | Done | Two repos established, Standard Product Taxonomy enabled, mirror tables in Supabase, Ecomus archived |
| 1 | Catalog Taxonomy Design | Done | Incl. 1d (Tag→Smart Collection); work landed in [[Phase 1 — Catalog Model]] |
| 2 | Catalog Data Migration | Done | 1,209 products + Brand metaobjects + metafield defs live on Shopify; see [[Phase 2 — Catalog Data Migration]] |
| 3 | Pages, Redirects, SEO Plumbing | Done | 1,512 redirects (99.5% WC sitemap coverage, audited 2026-05-23), pages migrated, collection_type metafield on all 463 collections. See [[Phase 3 — Pages, Redirects, SEO]] |
| 4 | Customers Migration | Done | 11 customer accounts imported (CSV, no order history). See [[Phase 4 — Customers Migration]] |
| 5 | Storefront Theme (Hyper) | Done | Hyper live; theme work complete, password gate removed 2026-05-24. See [[Phase 5 — Storefront Theme (Hyper)]] |
| 5b | Linked Products (post-launch) | Not Started | Pattern 3 Metaobject Product Group; Wishlist/Compare/Quick View moved into Phase 5 (Hyper toggles) |
| 6 | Sales Channels | Done | Google & YouTube + Shop App live. FB/IG + Pinterest deferred until catalog enriched. See [[Phase 6 — Sales Channels]] |
| 7 | Shipping & Payments | Done | Shopify Payments live (Stripe disabled), shipping config + page live, Afterpay under review. See [[Phase 7 — Shipping & Payments]] |
| 8 | Cutover | Done | DNS flipped 2026-05-24. www.techloop.com.au primary, .myshopify auto-301s. Apex cert provisioning in flight. See [[Phase 8 — Cutover]] |
| 9 | Workflow Rebuild Tier 1 — Inventory | ✅ Done (2026-05-30) | **TL_Mirror_Shopify** (`B6BBVJkRRCZP69HW`) + **TL_Inventory_Syncer_Shopify** (`uQ6XmGi3mspxKD56`) both **live**. Syncer first full run: 387 synced (238 corrections + 149 zero-outs), 0 failed; controlled 5-product test verified on live Shopify. Prerequisite **TL_Ingest_Leader_Feed reactivated** (`mUkhS0BfN7Lq6EsS`). Scope = Mirror + Syncer only (Detector → Phase 10). See [[Phase 9 — Workflow Rebuild Tier 1 (Inventory)]] |
| 10 | Workflow Rebuild Tier 2 — Onboarding | ✅ Done (2026-05-31) | **TL_Product_Detector_Shopify** (`fz50fq1dCmrHhazX`, active) + **TL_Product_Publisher_Shopify** (`pgWWBz9f6RXLXEIe`, built+verified, **inactive** pending go-live) + **TL_Slack_Interaction_Handler_Shopify** (`hikIeVV081e76pEv`, active). Category map fully resolved (426 approved / 0 pending); onboarding view yields **771 products**. Only remaining = flip Publisher active. See [[Phase 10 — Workflow Rebuild Tier 2 (Onboarding)]] |
| 11 | Workflow Rebuild Tier 3 — Enrichment | Not Started | TL_Enrich_Attributes_Shopify → metafields |
| 12 | Workflow Rebuild Tier 4 — Alerts/Utilities | Not Started | TL_Queue_Reviewer, TL_Price_Watchdog, etc. |
| 13 | Decommission WooCommerce | Not Started | Take WC offline once all workflows green |
| 14 | Shopify-native AI integrations | Not Started | Post-launch MCP connectors, AI taxonomies |

### ⚠️ Prerequisite (gap found 2026-05-30) — Reactivate frozen "no change" upstream workflows

The migration plan classified 6 upstream workflows as **"No change"** (they write to Supabase only, no platform rewrite needed) — but **never tasked reactivating them** after the 2026-05-05 blanket freeze. Result: the entire automation suite sat off, and the Leader feed went ~27 days stale. This is a **prerequisite (Tier 0)** for the workflow-rebuild phases, not a standalone phase.

| Upstream workflow | Needed by | Status |
|---|---|---|
| `TL_Ingest_Leader_Feed` (`mUkhS0BfN7Lq6EsS`) | Phase 9 (Syncer), Phase 10 (Detector) | ✅ **Reactivated 2026-05-30** (renamed, `0 */2 * * *`, feed refreshed: 19,628 live products) |
| `TL_Slack_Interaction_Handler` (`rDloLTYACT56kfpn`) | Phase 10 (HITL approve/reject) | ✅ **Superseded 2026-05-31** by Shopify clone `TL_Slack_Interaction_Handler_Shopify` (`hikIeVV081e76pEv`, active); WC original stays frozen |
| `TL_URL_Discoverer`, `TL_Scraper`, `TL_Enrich_Attributes`, `TL_Attribute_Proposer`, `TL_Enrichment_Reviewer` | Phase 11 (Enrichment) | Frozen — reactivate + verify at Phase 11 |
| `TL_Queue_Reviewer` | Phase 12 (Utilities) | Frozen — reactivate at Phase 12 |

Tracked as a Notion task in Tasks (Engineering). Each reactivation needs a verify step (feed source/credentials still valid, schedule re-registered) since they've been off ~1 month.

### Live store snapshot (verified via Admin GraphQL, 2026-05-23)

- **Products:** 1,209 — all ACTIVE (0 draft). Storefront password-gate state not exposed via Admin API; needs UI check before Phase 8.
- **Collections:** **463** total = 71 manual + 392 smart. The jump from the 2026-05-20 figure (337) is the brand-collection backfill: all 124 WC brands now have brand smart collections (was 52). `custom.collection_type` metafield set on 100% (244 category, 124 brand, 52 series, 32 attribute, 6 brand+category, 4 special, 1 tag).
- **Brand metaobject:** **125** entries (type `brand`) — all WC brands + Micron parent. (Memory previously recorded 124; live is 125.)
- **URL redirects:** **1,512** (incl. 124 brand redirects; +4 added 2026-05-23 in the Phase 3 audit). 99.5% WC-sitemap coverage.
- **Customers:** 11 (Phase 4 import; small base).
- **Theme (Main / Live):** **FoxEcom Hyper v1.3.3** (themeStoreId 3247), renamed "TechLoop". Theme detail owned by the active Phase 5 session. (Theme name/version not re-queryable here — token lacks `read_themes`.)
- **Locations:** 5 (NSW, QLD, SA, VIC, WA). NSW only stocked: 711 products, 10,962 units (per Phase 2 close; not re-queried 2026-05-23 — token lacks `read_inventory`).
- **Image coverage:** 1,208 / 1,209 (Ubiquiti UniFi G5 Turret Ultra wc_id 24403 → empty WC gallery → deferred to Phase 9 Crawl4AI enrichment).
- **Shop plan:** Basic, currency AUD.

### Repos, dev store, and domains

- **Long-lived repo:** `~/Code/techloop-shopify/` (theme at `theme/`, custom app, MCP integrations).
- **Throwaway migration repo:** `~/Code/techloop-migration/` (one-off WC→Shopify scripts).
- **Permanent myshopify domain:** `zc30tg-fi.myshopify.com` — **use this for Shopify CLI auth (`shopify store auth --store zc30tg-fi.myshopify.com …`) and any new API integrations.** The vanity alias errors out OAuth callbacks.
- **Primary alias:** `techloop-7.myshopify.com` — what users see in admin / preview URLs.
- **Theme branch:** `theme/hyper-base` → tag `theme/hyper-v1.0` at Phase 5.I activation.
- **Open items not yet resolved:** see Section 12.

---

## 1. Why Migrate

### Primary Driver — Infrastructure Overhead
The self-hosted WooCommerce stack requires constant attention to server security — bot mitigation, CloudFlare rules, DDoS protection. This overhead takes time away from building the actual store. Shopify handles all of this natively.

### Secondary Drivers
- **Shopify's platform direction** — AI-first features, native MCP connectors and APIs that make automation simpler. WooCommerce has been working towards similar capabilities but hasn't delivered.
- **Mobile-first management** — The goal is to run the store from a phone, largely from Slack. Shopify's mobile app and Slack integrations support this well.
- **Focus** — Stop fighting infrastructure, start building a store.

---

## 2. Business Context

- **Store type:** Drop shipping (tech / electronics / gaming)
- **Suppliers:** One supplier connected (Leader), others planned as future projects
- **Automation:** Heavily automated via n8n — feed ingestion, product detection, AI enrichment, inventory sync, HITL approval via Slack
- **Current state:** Side hustle — limited time invested in storefront due to infrastructure battles
- **Revenue model:** Margin on supplier cost (DBP) vs RRP

---

## 3. Build vs Buy Philosophy

**Default: build it.** With AI and Claude Code, building simple integrations is preferable to paying for apps — full control, no recurring costs, no vendor lock-in.

**Exception: official integrations** where the app provides exclusive API access or connections that would otherwise require applications/approvals (e.g., Google Shopping integration with supplementary feeds, Facebook sales channel).

**Rule of thumb:** If the cost-benefit of building doesn't stack up, or if an official connection provides capabilities you can't replicate, use the app.

---

## 4. Product Structure

### 4.1 Categories / Collections

**Current WooCommerce approach:**
- Hierarchical taxonomy — e.g., Computers → Components → Motherboards → (by chipset)
- Categories are singular in nature (one axis per level)
- Deliberately avoids combined attribute+category terms (e.g., no "Red Cases" — that's a colour attribute + case category)

**How Shopify handles this:**
- **Collections are flat** — no parent/child hierarchy. Two types: manual (hand-picked) and smart (rule-based auto-population with up to 60 conditions).
- **Smart collection rules** can filter by: title, type, vendor, tag, price, inventory, metafield values, and crucially **product category with descendants** (`PRODUCT_CATEGORY_ID_WITH_DESCENDANTS`).
- **Shopify Standard Product Taxonomy** is a separate, hierarchical category system (like Google Product Taxonomy). Each product gets one category. It maps natively to Google Product Categories for shopping feeds. Shopify Magic (AI) auto-suggests categories based on product data.
- **Navigation menus** provide visual hierarchy (up to 3 levels deep), linking to flat collections. This is how merchants simulate deep taxonomies.
- **No limit** on how many collections a product can belong to (store limit: 5,000 smart + ~100,000 manual collections).

**Recommended approach for TechLoop:**
- Assign each product a category from Shopify's Standard Product Taxonomy (maps to Google taxonomy automatically).
- Create flat collections for browsable groups (e.g., "Motherboards", "AMD Motherboards").
- Use smart collections with `PRODUCT_CATEGORY_ID_WITH_DESCENDANTS` for broad umbrella collections (e.g., "All Computer Components").
- Build the visual hierarchy through nested navigation menus (up to 3 levels).
- Lean on filtering (by metafields/attributes) for deeper drill-down within collections, rather than creating hyper-specific collections.

> **Decision needed:** How granular should collections be vs relying on filtering? E.g., one "Motherboards" collection with chipset filters, or separate "AMD Motherboards" / "Intel Motherboards" collections?

### 4.2 Attributes

**Current WooCommerce approach:**
- Global attributes with terms (e.g., "Chipset" attribute with "LGA 1700", "AM5" terms)
- Per-product attribute assignment
- Full enrichment pipeline: LLM extraction → HITL review → WC push
- Some attributes have their own archive pages (e.g., Series like "TUF Gaming", compatibility like "Works with Apple HomeKit")

**How Shopify handles this:**
- **No equivalent of WC global attributes.** Shopify uses **metafields** (key-value pairs with typed definitions) for product specs.
- **Category metafields** — Shopify's Standard Product Taxonomy auto-suggests relevant attributes per category (e.g., categorise as "Laptops" and get screen size, RAM, processor as suggested metafields). These are AI-driven via Shopify Magic.
- **Metafield definitions** formalise the schema: namespace, key, type, validation rules, storefront visibility. ~60 data types supported including measurements (dimension, weight, frequency, voltage, data storage capacity, etc.) — well-suited for tech specs.
- The existing enrichment pipeline (scraping → LLM extraction → HITL review) continues to work — it just pushes to metafields instead of WC global attributes.
- **Attribute archive pages** (e.g., "TUF Gaming" series page, "Works with Apple HomeKit" page) — can be modelled as smart collections filtered by a metafield value, or as metaobject pages.

**Filtering — native, no paid app required:**
- Shopify's free **Search & Discovery app** (first-party, by Shopify) manages storefront filters.
- Supports filtering by: availability, price, product type, vendor, tags, category, **product options**, **product metafields**, **variant metafields**, and **metaobject references**.
- Metafield types supported for filtering: `single_line_text_field`, `list.single_line_text_field`, `metaobject_reference`, `list.metaobject_reference`, `number_integer`, `number_decimal`, `boolean`.
- **Limit: 25 filters per store.** Filter logic: AND between filters, OR within a filter.
- Filters hidden if collection exceeds 5,000 products.
- Requires an **Online Store 2.0 theme** — themes render filters via `collection.filters` Liquid objects with full UI control.

> **This is good news** — no third-party filter plugin needed. Metafield-based filtering is native and free. The enrichment pipeline just needs to target metafields instead of WC attributes, and those metafields automatically become available as filters.

### 4.3 Brands

**Current WooCommerce approach:**
- Custom taxonomy for brands
- Some brands have sub-brands (e.g., ASUS → ROG, ASUS → Aurora; Micron → Crucial)
- `tl_brand_map` table maps supplier brand names to WC brand terms

**How Shopify handles this:**
- Shopify has a native **`vendor` field** on every product — free-text string, filterable, appears in admin and storefront. Good enough for basic brand assignment.
- For a richer brand experience (logos, descriptions, brand pages), use **metaobjects**: create a "Brand" metaobject definition with name, logo, description, website URL fields. Link products via a `metaobject_reference` metafield.
- **Sub-brands:** No native hierarchy. Model via a **self-referencing metaobject** — create a "Parent Brand" field on the Brand metaobject that references another Brand. E.g., "ROG" metaobject has parent = "ASUS". Requires custom Liquid in the theme to traverse the hierarchy.
- Metaobject entries can have their own pages on the storefront — so brand landing pages are possible natively.

**Recommended approach for TechLoop:**
- Use `vendor` field for the primary brand name (simplest, works with smart collections and filters).
- Create Brand metaobjects for brands needing pages/logos/sub-brand relationships.
- `tl_brand_map` continues to work — maps supplier brand to Shopify vendor value and/or metaobject reference.

### 4.4 Variants vs Individual Products

**Current WooCommerce approach:**
- Deliberate decision: **no product variants** — each SKU is an individual product
- Products in the same series are linked via a "linked products" plugin
- Variations happen across: memory capacity, CAS latency, colour, form factor (e.g., tenkeyless), switch type, etc.

**Reasoning:**
- Individual products get their own pages — better for search since people search for specific SKUs
- Products in a "series" can be quite loosely connected
- Simpler to automate (one product = one record)

**Why not Shopify variants?**
- Variants share one SEO title, one meta description, one product description, and one URL (just `?variant=123456`). Google indexes them as a single page.
- For electronics where a 16GB kit and a 64GB kit target different buyers and different search queries, separate products give separate organic search presence.
- The 3-option limit is also restrictive for loosely-connected series.
- Combined Listings (Plus only) uses a parent-child model which is conceptually wrong — series products are siblings, not children of an abstract parent. Plus is not on the cards.

**MVP decision: keep individual products** (one SKU = one product) — correct for SEO, search, and automation simplicity. This is what's live on Shopify today.

**Linking siblings is deferred to Phase 5b (post-launch).** The planned approach is the Metaobject "Product Group" pattern: each product family becomes a `Product Group` metaobject (with `name`, `group_option`, and a `list.product_reference` of siblings); each product gets `custom.product_group` (metaobject_reference) and `custom.option_value`; theme Liquid reads the group and renders swatch buttons. See [[Shopify Linked Products Research#Recommended]] for full pattern detail. **Not blocking go-live** — products can be linked retrospectively.

---

## 5. Content to Migrate

### Products
- All active products with: titles, descriptions, images, pricing, SKUs, attributes, brand assignments, category assignments
- Supplier metadata (ACF fields → Shopify metafields)
- Processed images (square format with white padding, hosted in WP media library)

### Categories → Collections
- Full hierarchical taxonomy needs mapping to Shopify's collection model
- `tl_category_map` table needs updating with Shopify collection IDs

### Brands
- All brands including sub-brand relationships

### Pages
- Warranties, support, FAQs, and similar informational pages
- Happy to rethink these to fit Shopify's structure — they don't need to be 1:1 replicas
- No blogs to migrate (was always a future piece of work)

### Google Shopping Feed
- **Crucial** — currently active, needs to be migrated.
- **Shopify's Google & YouTube channel** is free (first-party app). Syncs product catalogue to Google Merchant Center automatically. Enables free product listings across Google Shopping, Search, Images, and Lens.
- **Supplementary feeds:** You can create supplementary feeds in GMC (e.g., via Google Sheets) to add extra attributes like `product_highlight` or custom labels. However, fields synced via Content API **cannot be reliably overridden** manually in GMC — supplementary feeds can add new attributes but not override existing API-synced fields.
- Shopify's Standard Product Taxonomy maps to Google Product Categories natively — no manual GPC mapping needed.
- If supplementary feed control becomes a blocker, third-party feed apps (Simprosys, AdNabu, DataFeedWatch) offer more granular control.

### Other Sales Channels
- **Facebook & Instagram by Meta** — free first-party Shopify app. Syncs catalogue to Meta Commerce Manager, handles Pixel + Conversions API setup. As of Aug 2025, checkout redirects to Shopify storefront (not on-platform).
- Both Google and Meta channels are official integrations — aligns with the "use official where it gives exclusive access" philosophy.

---

## 6. What Doesn't Need to Migrate

- **Reviews** — none exist on the current store
- **Blogs** — none exist
- **Order history** — TBD, evaluate necessity
- **Customer accounts** — TBD, evaluate current customer base size and value

---

## 7. Secondary / Nice-to-Have (Not MVP-Blocking)

| Feature | Notes | Shopify Support |
|---------|-------|-----------------|
| **Wishlist** | Would be nice to have, not blocking for launch | **No native feature.** Requires an app — free options exist (Swym Wishlist Plus, Smart Wishlist). Or build custom via metafields + customer accounts. |
| **Product compare** | Side-by-side product comparison, not blocking | **No native feature.** Requires an app (Equate, Comparely) or custom build. Could build with Liquid + JS using metafield data. |
| **Multi-location inventory** | Suppliers have multiple warehouses around the state. More accurate shipping times. Not blocking. | **Natively supported.** 10 locations on Basic/Grow/Advanced plans. Order routing by proximity/priority. Stock transfers between locations supported. Shipping profiles can set rates per location group. |
| **Shipping optimisation** | Current rates are uncomplicated. Explore native integration to avoid margin erosion. | Shopify supports flat rates, carrier-calculated rates (Australia Post, etc.), and per-location-group shipping zones. |
| **Automated category creation** | Build a system that identifies keyword opportunities and creates collections. | Smart collections via API — can be automated through n8n. |

---

## 8. SEO Considerations

- 301 redirects from all WooCommerce URLs to Shopify equivalents — **critical**
- URL structure will change (WC `/product/slug` → Shopify `/products/handle`)
- Google Shopping feed continuity
- Existing Rank Math SEO data (meta titles, descriptions) needs to carry across
- Attribute/category archive pages that currently rank may need equivalent Shopify pages

---

## 9. Shopify Theme

**Resolved: FoxEcom Hyper v1.3.3.** Purchased 2026-05-16 via Shopify Theme Store (themeStoreId 3247, sold by FoxEcom). Installed on the live store, renamed "TechLoop", set as Main theme.

**Approach (locked 2026-05-16):** import a Hyper demo + rebrand with the Claude design system. The clean-room reimplementation that was locked 2026-05-12 was abandoned same day — too long. All 18 clean-room sub-tickets archived; replaced with 9 new Hyper sub-tasks 5.A–5.I.

- **Local repo:** `~/Code/techloop-shopify/theme/`
- **Branch:** `theme/hyper-base` → tag `theme/hyper-v1.0` at Phase 5.I activation
- **Hyper docs:** https://docs.foxecom.com/hyper-theme
- **Claude design system:** https://api.anthropic.com/v1/design/h/D6C4vinwsadVd9ViZtY3uQ — implement `ui_kits/store/index.html`
- **Wishlist / Compare / Quick View:** native Hyper toggles (no separate apps)
- **Previous WC theme work (Ecomus):** killed; was WP-specific

See [[Phase 5 — Storefront Theme (Hyper)]] for the sub-task tree (5.A–5.I), Definition of Done, and model/sub-agent strategy.

---

## 10. Automation Migration

See [[Shopify Migration Considerations]] for the full technical breakdown. Summary:

| Impact | Workflows |
|--------|-----------|
| **No change** | TL_Ingest_Leader_Feed, TL_URL_Discoverer, TL_Scraper, TL_Enrich_Attributes, TL_Attribute_Proposer, TL_Enrichment_Reviewer |
| **Minor changes** | TL_Queue_Reviewer, TL_Product_Detector, TL_Price_Watchdog |
| **Major rewrite** | TL_Mirror_WooCommerce, TL_Inventory_Syncer, TL_Product_Publisher, TL_Attribute_WC_Pusher, TL_Mirror_WC_Attributes |
| **3 button handlers** | TL_Slack_Interaction_Handler (publish, new attribute, new term) |

**Key architectural advantage:** The Supabase middleware layer means most workflows don't talk to WooCommerce directly. The platform-specific layer is isolated to ~5 workflows.

---

## 11. Shopify Plan Pricing (AUD, 2026)

| Plan | Monthly | Annual (per month) | Staff Accounts | Card Rate (Shopify Payments) |
|------|---------|--------------------|----------------|------------------------------|
| **Starter** | $11 | — | — | Social selling only |
| **Basic** | $56 | $42 | 1 | 2.9% + 30c |
| **Grow** (formerly "Shopify") | $149 | ~$114 | 6 | 2.7% + 30c |
| **Advanced** | $575 | ~$431 | 16 | 2.4% + 30c |
| **Plus** | ~$3,700 | — | Unlimited | Negotiable |

All prices AUD. 10% GST added on top. Shopify bills in USD with exchange rate conversion.

---

## 12. Outstanding Questions

Resolved items (Shopify plan, cutover model, storefront approach, wishlist/compare, customer migration scope, image source policy, single-location inventory pattern) have been moved into the Status section + relevant body sections. Only genuinely open items remain.

### Open

- **Shopify Payments vs Stripe** (Phase 7) — Shopify Payments removes the 0.5–2% third-party-gateway fee but locks us into Shopify's processor rates. Decision pending fee modelling.
- **Timeline expectations** — no external deadline locked in.
- **Canonical / SEO during pre-cutover gap** — password gate was removed 2026-05-24 but `techloop.com.au` still points to WC. Verify the Shopify storefront has `noindex` (or canonical tags pointing back to techloop.com.au) until DNS flips, to avoid Google indexing duplicate content at `techloop-7.myshopify.com`.
- **Meta titles/descriptions QA** (Phase 3 remaining) — confirm Rank Math meta data carried across to Shopify SEO fields at scale.
- **Electronics (WC 44) category mapping** — auto-recommendation was no-match (top-level taxonomy node `gid://shopify/TaxonomyCategory/el`). Deferred to Phase 2 Step 05 second-pass enrichment; still open.

### Resolved 2026-05-23

- **Phase 3 status conflict** — RESOLVED. Live store has 1,512 redirects (99.5% sitemap coverage) + collection_type metafield + migrated pages. Phase 3 marked **Done**; see [[Phase 3 — Pages, Redirects, SEO]].
- **Collection count reconciliation** — RESOLVED. Live = **463** (71 manual + 392 smart). The earlier 280/334/337 figures predate the brand-collection backfill (all 124 WC brands → brand smart collections). collection_type metafield distribution accounts for all 463.
- **5 unresolved Step 06 collections** — Gaming Eyewear (0p, manual), Party Speakers (3p), Memory umbrella (no clean parent — multiple split collections instead), Intel LGA1851 (handle resolved as `motherboards-intel-lga1851-15th-gen`), Camera Mounts/Grips/Stabilizers (0p, title has HTML-encoded `&amp;`). Need final disposition.
- **techloop-7.myshopify.com vs zc30tg-fi.myshopify.com** — vault + Notion docs use the alias `techloop-7`; permanent domain is `zc30tg-fi`. Worth a search/replace pass for any API-context references.
