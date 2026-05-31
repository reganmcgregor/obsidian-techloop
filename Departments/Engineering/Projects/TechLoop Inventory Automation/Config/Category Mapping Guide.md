# Category Mapping Guide

**Last Updated:** 2026-02-02

---

## Overview

Category mapping is essential for the product onboarding pipeline. It determines:
- Which WooCommerce category new products are assigned to
- Whether products should be blocked or auto-approved
- The default margin percentage for pricing

Without a category mapping, products will be queued for review but **cannot be published** until mapped.

### Subcategory-Level Mapping

Mappings support **subcategory-level precision**. The system matches:
1. First by `category_code` + `subcategory_name` (most specific)
2. Falls back to `category_code` only if no subcategory match exists

This allows granular control, e.g.:
- `CB > Network Cables` → WC "Network Cables" (mapped)
- `CB > Fibre` → Blocked (different subcategory treatment)

---

## Table: tl_category_map

| Column | Purpose |
|--------|---------|
| `supplier_code` | Supplier identifier (default: 'LEADER') |
| `supplier_category_code` | Leader category code (e.g., 'CB', 'NB') |
| `supplier_category_name` | Human-readable category name |
| `supplier_subcategory_name` | Subcategory for granular mapping (e.g., 'Network Cables') |
| `wc_category_id` | WooCommerce category term_id |
| `wc_category_name` | WooCommerce category name (for reference) |
| `is_blocked` | If true, products are auto-rejected |
| `block_reason` | Why blocked |
| `auto_approve` | If true, products skip review |
| `default_margin_percent` | Default markup % |

**Unique constraint:** `(supplier_code, supplier_category_code, supplier_subcategory_name)`

---

## Finding Unmapped Categories

Use the `vw_unmapped_categories` view to find Leader categories that need mapping:

```sql
SELECT
    category_code,
    category_name,
    product_count,
    in_stock_count
FROM vw_unmapped_categories
ORDER BY product_count DESC;
```

This shows:
- Categories with products in the Leader feed
- How many products are in each category
- How many are currently in stock

---

## Adding a Category Mapping

### Step 1: Find the WooCommerce Category ID

In WooCommerce admin:
1. Go to Products > Categories
2. Click on the category you want to map to
3. Look at the URL - the ID is the `tag_ID` parameter

Or query the database:
```sql
SELECT term_id, name
FROM wp_terms t
JOIN wp_term_taxonomy tt ON t.term_id = tt.term_id
WHERE tt.taxonomy = 'product_cat'
ORDER BY name;
```

### Step 2: Insert the Mapping

```sql
INSERT INTO tl_category_map (
    supplier_code,
    supplier_category_code,
    supplier_category_name,
    wc_category_id,
    wc_category_name,
    auto_approve,
    default_margin_percent
) VALUES (
    'LEADER',
    'HARDWARE > NOTEBOOKS',
    'Notebooks',
    155,
    'Laptops',
    false,
    18.00
);
```

---

## Common Mappings

Here are suggested mappings for common Leader categories:

| Leader Category | WC Category | Margin | Auto-Approve |
|-----------------|-------------|--------|--------------|
| HARDWARE > NOTEBOOKS | Laptops | 18% | No |
| HARDWARE > DESKTOPS | Desktop PCs | 15% | No |
| HARDWARE > MONITORS | Monitors | 20% | No |
| PERIPHERALS > KEYBOARDS | Keyboards | 25% | Yes |
| PERIPHERALS > MICE | Mice | 25% | Yes |
| NETWORKING > ROUTERS | Networking | 22% | No |
| STORAGE > SSD | Storage | 20% | Yes |
| STORAGE > HDD | Storage | 20% | Yes |

---

## Blocking Categories

Some categories may not be suitable for your store. Block them to prevent products from entering the queue:

```sql
INSERT INTO tl_category_map (
    supplier_code,
    supplier_category_code,
    supplier_category_name,
    is_blocked,
    block_reason
) VALUES (
    'LEADER',
    'ACCESSORIES > CABLES',
    'Cables',
    true,
    'Low margin, high competition'
);
```

Blocked categories:
- Products are detected but marked as `status = 'blocked'`
- They do not appear in review queue
- Can be unblocked later if needed

---

## Auto-Approve Categories

For trusted categories with consistent quality, enable auto-approval:

```sql
UPDATE tl_category_map
SET auto_approve = true
WHERE supplier_category_code = 'PERIPHERALS > KEYBOARDS';
```

Auto-approved products:
- Skip the `pending_review` status
- Go directly to `approved` status
- Will be published on next workflow run

**Use with caution** - only for categories you trust completely.

---

## Adjusting Margins

Default margin determines the selling price:

```
Selling Price = DBP Inc GST * (1 + margin_percent / 100)
```

Update margin for a category:

```sql
UPDATE tl_category_map
SET default_margin_percent = 22.00
WHERE supplier_category_code = 'HARDWARE > MONITORS';
```

### Margin Guidelines

| Category Type | Suggested Margin | Reason |
|---------------|------------------|--------|
| Laptops | 15-18% | High competition, price-sensitive |
| Monitors | 18-22% | Moderate competition |
| Peripherals | 22-28% | Lower price point, more margin |
| Accessories | 25-35% | Impulse purchases |
| Networking | 20-25% | Technical products |
| Gaming | 18-22% | Enthusiast market |

---

## Bulk Import Mappings

For initial setup, you can bulk import mappings:

```sql
INSERT INTO tl_category_map (
    supplier_code,
    supplier_category_code,
    supplier_category_name,
    wc_category_id,
    wc_category_name,
    auto_approve,
    default_margin_percent
) VALUES
    ('LEADER', 'HARDWARE > NOTEBOOKS', 'Notebooks', 155, 'Laptops', false, 18.00),
    ('LEADER', 'HARDWARE > DESKTOPS', 'Desktops', 156, 'Desktop PCs', false, 15.00),
    ('LEADER', 'HARDWARE > MONITORS', 'Monitors', 157, 'Monitors', false, 20.00),
    ('LEADER', 'PERIPHERALS > KEYBOARDS', 'Keyboards', 160, 'Keyboards', true, 25.00),
    ('LEADER', 'PERIPHERALS > MICE', 'Mice', 161, 'Mice', true, 25.00),
    ('LEADER', 'NETWORKING > ROUTERS', 'Routers', 170, 'Networking', false, 22.00)
ON CONFLICT (supplier_code, supplier_category_code) DO UPDATE
SET
    wc_category_id = EXCLUDED.wc_category_id,
    wc_category_name = EXCLUDED.wc_category_name,
    auto_approve = EXCLUDED.auto_approve,
    default_margin_percent = EXCLUDED.default_margin_percent,
    updated_at = NOW();
```

---

## Viewing Current Mappings

### All Mappings

```sql
SELECT
    supplier_category_code,
    supplier_category_name,
    wc_category_id,
    wc_category_name,
    is_blocked,
    auto_approve,
    default_margin_percent
FROM tl_category_map
WHERE supplier_code = 'LEADER'
ORDER BY supplier_category_code;
```

### Active (Not Blocked)

```sql
SELECT *
FROM tl_category_map
WHERE supplier_code = 'LEADER'
  AND is_blocked = false
ORDER BY supplier_category_code;
```

### Blocked Categories

```sql
SELECT
    supplier_category_code,
    supplier_category_name,
    block_reason
FROM tl_category_map
WHERE supplier_code = 'LEADER'
  AND is_blocked = true;
```

### Auto-Approve Categories

```sql
SELECT
    supplier_category_code,
    supplier_category_name,
    default_margin_percent
FROM tl_category_map
WHERE supplier_code = 'LEADER'
  AND auto_approve = true;
```

---

## Per-Product Category Overrides

When category mappings are too broad, you can override the WooCommerce category for individual products in the onboarding queue **before** publishing.

### Why Override in Queue (Recommended)

- Product gets the **correct URL slug from the start** (WordPress generates slug at creation)
- Less manual work after publishing
- Keeps WooCommerce store cleaner

### How to Override

**Via SQL:**
```sql
UPDATE tl_onboarding_queue
SET mapped_wc_category_id = 1234,
    mapped_wc_category_name = 'Specific Category Name'
WHERE stock_code = 'ABC123';
```

**Via Supabase Table Editor:**
1. Open `tl_onboarding_queue` table
2. Find the product by `stock_code`
3. Edit `mapped_wc_category_id` field
4. Save changes
5. Set status to `approved`

The Product Publisher will use the overridden category when creating the product.

---

## Troubleshooting

### Products Not Appearing in Queue

**Possible causes:**
1. Category is blocked - check `is_blocked` flag
2. No mapping exists - check `vw_unmapped_categories`
3. Product already in WooCommerce - check `tl_wc_products_mirror`
4. Product already in queue - check `tl_onboarding_queue`

### Products Stuck in pending_review

**Check:**
1. Is there a category mapping with `wc_category_id`?
2. Products cannot publish without a valid WC category

### Wrong Prices

**Check:**
1. `default_margin_percent` for the category
2. Price is calculated as: `dbp_inc_gst * (1 + margin/100)`

---

## Maintenance

### Regular Review

Monthly, review:
1. New unmapped categories: `SELECT * FROM vw_unmapped_categories`
2. Queue status: `SELECT * FROM vw_onboarding_queue_summary`
3. Blocked categories - still relevant?

### Updating WooCommerce Category IDs

If WooCommerce categories are reorganised:

```sql
-- Update mapping with new category ID
UPDATE tl_category_map
SET wc_category_id = 200,
    wc_category_name = 'New Category Name'
WHERE supplier_category_code = 'HARDWARE > NOTEBOOKS';
```

---

## Related Documentation

- [[Workflow 5 - Product Detector]]
- [[Workflow 6 - Product Publisher]]
- [[Database Schema - Product Automation]]
