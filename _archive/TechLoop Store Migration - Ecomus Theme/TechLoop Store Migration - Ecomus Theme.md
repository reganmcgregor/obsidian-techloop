---
status: in-progress
branch: TechLoop-Ecomus-POC
started: '2026-02-28'
tags: project,ecommerce,theme,woocommerce
notion_page_id: 3558c3d3-c03e-81b6-b0f8-c9f3e4e759a2
notion_parent_id: 7093b771-b742-41cd-9642-49e0b3c6531c
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-03T16:40:00Z
---
# TechLoop Store Migration — Ecomus Theme

## Overview

Full migration of the TechLoop WooCommerce store from the legacy Salient theme to the **Ecomus** premium ecommerce theme. This is a major overhaul — the entire store front-end is being replaced in one branch, with cleanup and polish to follow.

- **Branch:** `TechLoop-Ecomus-POC`
- **Status:** In progress
- **Local URL:** `https://local.techloop.com.au`

---

## Migration Scope

This is not an incremental change — the entire store is being re-themed at once. Key areas being changed:

- All front-end templates and layouts
- Product pages, archive/shop pages, cart, checkout
- Home page and landing pages
- Navigation, header, footer
- Child theme CSS/JS rebuilt from scratch for Ecomus

Post-migration cleanup tasks will be tracked separately.

---

## Theme Architecture

### TechLoop Child Theme (`wp-content/themes/techloop/`)

Active theme. All customisations live here, never in the parent.

| Asset | Source | Output |
|---|---|---|
| Styles | `assets/scss/style.scss` | `style.css` (theme root) |
| JavaScript | `assets/js/src/main.js` | `assets/js/dist/` |
| PHP customisations | `functions.php` | — |

**Build commands** (run from `wp-content/themes/techloop/`):

```bash
npm run dev    # BrowserSync + file watching
npm run build  # Production build
```

BrowserSync proxies `https://local.techloop.com.au` using SSL certs at `~/local.techloop.com.au*.pem`.

### Ecomus Parent Theme (`wp-content/themes/ecomus/`)

Premium theme — **do not edit directly**.

- Entry: `functions.php` → `inc/theme.php` → `inc/autoload.php`
- Autoloader: `Ecomus\Foo\BarBaz` maps to `inc/foo/bar-baz.php`
- Singleton pattern: `\Ecomus\Theme::instance()->init()`
- WooCommerce template overrides: `woocommerce/` directory
- Customizer: `inc/customizer.php` / Dynamic CSS: `inc/dynamic-css.php`
- WooCommerce logic: `inc/woocommerce/`

### Ecomus Addons Plugin (`wp-content/plugins/ecomus-addons/`)

Companion plugin bundled with the theme:

- Elementor widgets + page builder extensions (`inc/elementor/`)
- Custom Elementor template builder for shop/product/cart/checkout (`inc/elementor/builder/`)
- WooCommerce product enhancements (`inc/woocommerce/`)
- Admin backend settings (`inc/backend/`)

---

## Key Plugins

| Plugin | Purpose |
|---|---|
| WooCommerce | Core e-commerce |
| ecomus-addons | Theme companion — Elementor widgets, WC extensions |
| Elementor | Page builder |
| WP All Import Pro + WooCommerce Add-on | Product data imports |
| YITH Variation Swatches, Compare, Wishlist | Product UX |
| wcboost-products-compare, wcboost-wishlist | Additional product UX |
| Rank Math SEO Pro | SEO |
| WP Rocket | Caching |
| Imagify | Image optimisation |
| wp-migrate-db-pro | Database migration between environments |
| Stripe for WooCommerce | Payments |

---

## Decisions & Notes

| Date | Decision / Note |
|---|---|
| 2026-02-28 | Migration started on `TechLoop-Ecomus-POC` branch. Big-bang approach — full theme swap, cleanup to follow. |

---

## Related Notes

- [[TechLoop.md]] — Company overview & brand strategy
- [[System Overview - TechLoop Inventory Automation]] — Inventory automation project

#project #ecommerce #theme-migration #woocommerce
