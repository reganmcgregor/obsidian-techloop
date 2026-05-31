# Shopify Linked Products Research

> **Context:** In electronics retail, individual products (each with their own SKU, page, and URL) need to appear linked together on the product page as if they were variations — but they are genuinely separate products (siblings, not parent-child). For example, a RAM series in 16GB/32GB/64GB, keyboards in different switch types, or PC cases in different colours. In WooCommerce this is solved by a "linked products" plugin. This document covers all approaches for Shopify.

---

## Why Not Use Shopify Variants?

For electronics, **separate products is the correct approach.** Variants are for t-shirts in 3 colours, not RAM at different capacities with different CAS latencies.

| Factor | Variants | Separate Products |
|--------|----------|-------------------|
| Own SEO title/description | No — shared across all variants | Yes — individual per product |
| Own product description | No — one description for all | Yes |
| Own URL path | No — just `?variant=123456` | Yes — `/products/kingston-fury-32gb` |
| Google organic indexing | One listing for all variants | One listing per product |
| Google Shopping | Separate entries (works either way) | Separate entries |
| Full image gallery each | No — one image per variant | Yes |
| Automation complexity | Need to group into variant families | One product = one record (simpler) |
| Variant option limit | 3 options max | N/A |

**Key SEO facts:**
- Shopify variant URLs (`?variant=123456`) canonicalise back to the base product URL — Google indexes them as one page.
- Variants cannot have their own SEO meta title, meta description, or product description.
- Variant images are limited to one per variant (no full gallery).
- For products targeting different search queries (e.g., "32GB DDR5 RAM" vs "64GB DDR5 RAM"), separate products give separate organic search presence.

---

## Why Not Shopify Combined Listings (Plus)?

Combined Listings creates a **non-purchasable parent product** as a container for child products. This is the wrong model — electronics products in a series are **siblings**, not children of an abstract parent. Beyond the conceptual mismatch, it's gated behind Shopify Plus (~$3,700 AUD/month) and only works on the Online Store sales channel. Not viable.

---

## Approaches for Linking Separate Products

All approaches below keep individual products with their own URLs, SEO, descriptions, and images — and add visual linking on the storefront.

### Pattern 1: Tag-Based Grouping (Simplest)

Tag products with a shared group identifier, then use theme Liquid to render selector buttons.

**Setup:**
- Tag all products in a family with the same tag: `series:kingston-fury-ddr5-rgb`
- Theme Liquid collects products with the matching tag and renders buttons

**Automation:** Dead simple — n8n just adds a tag when creating/updating a product.

**Pros:**
- Minimal setup
- Easy to automate

**Cons:**
- Tags are flat strings — no way to specify *which attribute* the swatch label should show (Capacity vs Colour vs Size)
- Tag-based Liquid queries can be slow on large catalogues
- No structured data about the relationship

**Verdict:** Too limited for TechLoop's needs — different product families need different swatch labels.

---

### Pattern 2: Metafield `list.product_reference` + Option Labels

Link products via metafields with additional metadata for display.

**Metafield definitions on each product:**
- `custom.linked_products` — `list.product_reference` — the sibling products
- `custom.link_option_name` — `single_line_text_field` — e.g., "Capacity", "Colour", "Switch Type"
- `custom.link_option_value` — `single_line_text_field` — e.g., "32GB", "Black", "TKL"

**Theme Liquid:**
```liquid
{%- assign siblings = product.metafields.custom.linked_products.value -%}
{%- if siblings and siblings.size > 0 -%}
  <div class="product-siblings">
    <span class="option-label">{{ product.metafields.custom.link_option_name }}:</span>
    {%- comment -%} Current product shown as active {%- endcomment -%}
    <a href="{{ product.url }}" class="swatch active">
      {{ product.metafields.custom.link_option_value }}
    </a>
    {%- for sibling in siblings -%}
      <a href="{{ sibling.url }}" class="swatch">
        {{ sibling.metafields.custom.link_option_value }}
      </a>
    {%- endfor -%}
  </div>
{%- endif -%}
```

**Automation:** n8n uses `metafieldsSet` GraphQL mutation to set all three metafields when creating products.

**Pros:**
- Works on any Shopify plan, zero cost
- Handles different attribute types per group (Capacity for RAM, Colour for cases)
- Liquid resolves product references to full objects automatically — no N+1 query issues
- Full UI control

**Cons:**
- **Bidirectional linking required** — if A links to B and C, then B must also link to A and C, and C must link to A and B. Adding a product to a group means updating every existing sibling.
- Solvable via n8n (one `metafieldsSet` call updates all siblings in a batch of up to 25 metafields per call), but adds workflow complexity
- Page reloads on swatch click (unless JavaScript added for AJAX)

**Verdict:** Workable, but the bidirectional linking overhead makes Pattern 3 preferable.

---

### Pattern 3: Metaobject "Product Group" (Recommended) {#recommended}

Create a structured "Product Group" metaobject as the single source of truth. Products point to their group; the group knows its members.

**Metaobject definition — "Product Group":**

| Field | Type | Example |
|-------|------|---------|
| `name` | `single_line_text_field` | "Kingston Fury Beast DDR5 RGB" |
| `group_option` | `single_line_text_field` | "Capacity" |
| `products` | `list.product_reference` | [32GB product, 64GB product, ...] |

**Metafield definitions on each product:**

| Field | Type | Example |
|-------|------|---------|
| `custom.product_group` | `metaobject_reference` | → the Product Group metaobject |
| `custom.option_value` | `single_line_text_field` | "32GB" |

**Theme Liquid:**
```liquid
{%- assign group = product.metafields.custom.product_group.value -%}
{%- if group -%}
  <div class="product-siblings">
    <span class="option-label">{{ group.group_option }}:</span>
    {%- for sibling in group.products.value -%}
      {%- if sibling.id == product.id -%}
        <a href="{{ sibling.url }}" class="swatch active">
          {{ sibling.metafields.custom.option_value }}
        </a>
      {%- else -%}
        <a href="{{ sibling.url }}" class="swatch">
          {{ sibling.metafields.custom.option_value }}
        </a>
      {%- endif -%}
    {%- endfor -%}
  </div>
{%- endif -%}
```

**Why this is the best approach:**

1. **No bidirectional linking problem.** The metaobject is the single source of truth. Add a new product to the group by updating one metaobject — all siblings automatically see it.
2. **Handles different attributes per group.** The `group_option` field defines the swatch label: "Capacity" for RAM, "Colour" for cases, "Switch Type" for keyboards, "Size" for cases.
3. **Series landing pages for free.** Metaobjects can have their own storefront pages — a "Kingston Fury Beast DDR5 RGB" series page listing all capacities.
4. **Fully automatable via GraphQL API:**
   - `metaobjectCreate` / `metaobjectUpdate` to manage groups
   - `metafieldsSet` to link products to their group and set option values
5. **Zero cost** — native Shopify feature on all plans.
6. **Extensible.** Can add fields later: group image, group description, secondary option (e.g., Colour AND Capacity), display order.

**Multi-option groups (future):**
For products varying on two axes (e.g., RAM that varies by capacity AND colour), add a second option:
- `group_option_2` field on the metaobject: "Colour"
- `custom.option_value_2` metafield on each product: "Black", "White"
- Theme Liquid renders two rows of swatches

**n8n automation pattern:**
```
Leader Feed → Parse product titles/model series
  → Detect product families (matching base model, SKU prefix, or series name)
  → For each family:
    1. Check if Product Group metaobject exists (by name)
    2. If not → metaobjectCreate with group_option based on product category
       (RAM → "Capacity", Cases → "Colour/Size", Keyboards → "Switch Type")
    3. Add new product to the group's product list → metaobjectUpdate
    4. Set product's custom.product_group reference → metafieldsSet
    5. Set product's custom.option_value from Leader feed data → metafieldsSet
```

**Detecting product families from Leader feed:**
The Leader feed provides product titles and model numbers. Products in the same series typically share a base model name with a differentiating suffix. For example:
- `KF560C36BBEK2-32` and `KF560C36BBEK2-64` → same series, differs by capacity
- The LLM enrichment pipeline could extract "series" as an attribute during `TL_Enrich_Attributes`
- Alternatively, a rule-based approach: strip known differentiators (capacity, colour codes) from the model number and group by the remainder

---

### Pattern 4: Build a Custom Shopify App

Package the metaobject approach into a proper Shopify app with an admin UI and theme app extension.

**Architecture:**
- **Theme app extension** — an "app block" that renders the swatch UI inline on product pages. Merchants drag-and-drop it in the Theme Editor. Works on any OS 2.0 theme without editing theme code.
- **Admin UI** — Remix + Polaris app embedded in Shopify admin. Visual interface for creating/managing product groups with a resource picker for selecting products.
- **API** — n8n calls the Shopify GraphQL API directly (metaobjects + metafields). No separate API needed.

**Shopify app distribution options:**
- **Custom distribution** (own store only) — no App Store review needed, install via direct link
- **Public distribution** (App Store) — $19 USD one-time registration, 0% revenue share on first $1M USD annual revenue, 15% after that

**Effort estimate:**

| Scope | Timeline (solo dev) |
|-------|-------------------|
| MVP (functional) | 4–6 weeks |
| Polished (good UX, edge cases) | 8–10 weeks |
| App Store ready (docs, onboarding, billing) | 12+ weeks |

**Tech stack:** Remix (React), Polaris (Shopify design system), App Bridge, Shopify CLI, GraphQL Admin API, theme app extension (Liquid + JS/CSS).

**Verdict:** Worth considering as a future project and potential revenue stream. For MVP migration, start with Pattern 3 (raw metaobjects + theme Liquid) and package into an app later if it works well.

---

## Decision

**Pattern 3 (Metaobject "Product Group") is the chosen approach.**

- Zero cost, any plan, fully automatable
- No bidirectional linking overhead
- Handles different swatch labels per product category
- Can evolve into a sellable app later
- Products remain individual with their own SEO, descriptions, URLs, and images

**Implementation order:**
1. Create metaobject definition and metafield definitions in Shopify admin (or via API during store setup)
2. Add Liquid code to chosen theme's product template
3. Build n8n workflow for detecting product families and managing groups
4. Products can be uploaded first and linked later — grouping is non-blocking for go-live

---

## Sources

- [Shopify Dev — Metaobjects](https://shopify.dev/docs/apps/build/custom-data/metaobjects)
- [Shopify Dev — metaobjectCreate Mutation](https://shopify.dev/docs/api/admin-graphql/latest/mutations/metaobjectCreate)
- [Shopify Dev — metafieldsSet Mutation](https://shopify.dev/docs/api/admin-graphql/latest/mutations/metafieldsSet)
- [Shopify Dev — Liquid Metafield Object](https://shopify.dev/docs/api/liquid/objects/metafield)
- [Shopify Dev — Product Variant Object (Liquid)](https://shopify.dev/docs/api/liquid/objects/variant)
- [Shopify Dev — Product Variant Object (Storefront API)](https://shopify.dev/docs/api/storefront/latest/objects/ProductVariant)
- [Shopify Dev — Theme App Extensions](https://shopify.dev/docs/apps/build/online-store/theme-app-extensions)
- [Shopify Dev — App Blocks](https://shopify.dev/docs/storefronts/themes/architecture/blocks/app-blocks)
- [Shopify Dev — Combined Listings](https://shopify.dev/docs/apps/build/product-merchandising/combined-listings)
- [Shopify Dev — Custom App Distribution](https://shopify.dev/docs/apps/launch/distribution/select-distribution-method)
- [Shopify Dev — Revenue Share](https://shopify.dev/docs/apps/launch/distribution/revenue-share)
- [Shopify Dev — Build Apps with Remix](https://shopify.dev/docs/apps/build/build?framework=remix)
- [Shopify Help — Product SEO](https://help.shopify.com/en/manual/promoting-marketing/seo)
- [Google — Product Variants Structured Data](https://developers.google.com/search/docs/appearance/structured-data/product-variants)
