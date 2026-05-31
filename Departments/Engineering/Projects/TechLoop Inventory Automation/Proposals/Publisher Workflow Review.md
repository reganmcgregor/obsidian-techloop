# Publisher Workflow Review

**Purpose:** Review field mappings and workflow design before testing
**Status:** ✅ Decisions Confirmed (2026-02-03)

---

## Confirmed Field Mapping Decisions

### Core Product Fields

| WooCommerce Field | Source | Decision |
|-------------------|--------|----------|
| `name` | AI-optimised title | ✅ Use Ollama + RAG |
| `status` | Hardcoded `"draft"` | ✅ Correct |
| `type` | Hardcoded `"simple"` | ✅ Correct |
| `sku` | `manufacturer_sku` | ✅ Changed from stock_code |
| `regular_price` | `rrp` | ✅ Use RRP directly (pricing workflow later) |
| `description` | `product_name_2` | ✅ Map from Leader feed |
| `manage_stock` | `true` | ✅ Correct |
| `stock_quantity` | `availability_total` | ✅ From Leader feed |
| `stock_status` | Calculated | ✅ Based on availability |
| `categories` | `mapped_wc_category_id` | ✅ From category mapping |
| `images` | `image_url` | ✅ (Square optimisation deferred to separate workflow) |
| `weight` | `weight` | ✅ Both in kg, no conversion needed |
| `dimensions.length` | `length / 10` | ✅ Convert mm → cm |
| `dimensions.width` | `width / 10` | ✅ Convert mm → cm |
| `dimensions.height` | `height / 10` | ✅ Convert mm → cm |

### Product Attributes (Global)

| Attribute | WC ID | Source | Notes |
|-----------|-------|--------|-------|
| Brand | `106` | `manufacturer` | Global attribute |
| Model Number | `107` | `manufacturer_sku` | MPN |
| Manufacturer's Warranty | `173` | `warranty_length` | Format: `{months} Months` |

### Brand Taxonomy

| Taxonomy | Source | Notes |
|----------|--------|-------|
| `yith_product_brand` | `manufacturer` | Current brand plugin taxonomy |

**Note:** Brand attribute (id: 106) and yith_product_brand taxonomy are separate. The attribute is set automatically; the taxonomy requires brand mapping lookup.

**Future Migration:** Migrate from `yith_product_brand` to `product_brand` taxonomy.

### ACF/Meta Fields

| Meta Key | Source | Purpose |
|----------|--------|---------|
| `supplier_name` | Hardcoded `"Leader"` | Supplier identifier |
| `supplier_product_id` | `stock_code` | Leader SKU (for sync) |
| `supplier_dbp_ex_gst` | `dbp_ex_gst` | Cost ex GST |
| `supplier_dbp_inc_gst` | `dbp_inc_gst` | Cost inc GST |
| `supplier_mpn` | `manufacturer_sku` | Manufacturer part number |
| `supplier_barcode` | `bar_code` | EAN/UPC |
| `_wc_cog_cost` | `dbp_inc_gst` | WooCommerce Cost of Goods |
| `_wc_gtin` | `bar_code` | WooCommerce GTIN field |

### Product-Level Fields

| Field | Source | Purpose |
|-------|--------|---------|
| `global_unique_id` | `bar_code` | WC 8.x native GTIN |

### Fields NOT Mapped (By Decision)

| Leader Field | Decision | Reason |
|--------------|----------|--------|
| `category_name` | ❌ Don't map | Not needed |
| `subcategory_name` | ❌ Don't map | Not needed |

---

## Deferred to Separate Workflows

| Feature | Workflow | Notes |
|---------|----------|-------|
| Price Calculation | `TL_Price_Calculator` | Margins, sale prices, etc. |
| Image Optimisation | `TL_Image_Optimiser` | Square images, white bg extension |
| Brand Migration | `TL_Brand_Migration` | yith_product_brand → product_brand |
| Description Cleanup | (In Publisher) | Ollama AI to clean formatting |

---

## Brand Mapping

A `tl_brand_map` table has been created to map supplier brand names to WooCommerce brand taxonomy terms.

| Column | Purpose |
|--------|---------|
| `supplier_code` | Supplier identifier (default: 'LEADER') |
| `supplier_brand_name` | Brand name from supplier feed |
| `wc_brand_term_id` | WooCommerce yith_product_brand term ID |
| `wc_brand_name` | WooCommerce brand display name |
| `wc_brand_slug` | WooCommerce brand slug |
| `is_blocked` | If true, brand is excluded |

**Usage:** The Publisher workflow will lookup the brand mapping before creating products. If no mapping exists, it will create the brand term and add the mapping.

---

## Confirmed JSON Field Mapping

```json
{
  "name": "{{ AI_OPTIMISED_TITLE }}",
  "status": "draft",
  "type": "simple",
  "sku": "{{ manufacturer_sku }}",
  "regular_price": "{{ rrp }}",
  "description": "{{ product_name_2 }}",
  "short_description": "",
  "manage_stock": true,
  "stock_quantity": "{{ availability_total }}",
  "stock_status": "{{ availability_total > 0 ? 'instock' : 'outofstock' }}",
  "categories": [{ "id": "{{ mapped_wc_category_id }}" }],
  "images": [{ "src": "{{ image_url }}" }],
  "weight": "{{ weight }}",
  "dimensions": {
    "length": "{{ length / 10 }}",
    "width": "{{ width / 10 }}",
    "height": "{{ height / 10 }}"
  },
  "brands": [{ "id": "{{ brand_term_id }}" }],
  "attributes": [
    {
      "id": 0,
      "name": "Brand",
      "options": ["{{ manufacturer }}"],
      "visible": true
    },
    {
      "id": 0,
      "name": "Model Number",
      "slug": "model-number",
      "options": ["{{ manufacturer_sku }}"],
      "visible": true
    },
    {
      "id": 0,
      "name": "Manufacturer's Warranty",
      "slug": "manufacturers-warranty",
      "options": ["{{ warranty_length }} Months"],
      "visible": true
    }
  ],
  "meta_data": [
    { "key": "acf_supplier_name", "value": "Leader" },
    { "key": "acf_supplier_product_id", "value": "{{ stock_code }}" },
    { "key": "acf_supplier_dbp_ex_gst", "value": "{{ dbp_ex_gst }}" },
    { "key": "acf_supplier_dbp_inc_gst", "value": "{{ dbp_inc_gst }}" },
    { "key": "acf_supplier_mpn", "value": "{{ manufacturer_sku }}" },
    { "key": "acf_supplier_barcode", "value": "{{ bar_code }}" },
    { "key": "cost_of_goods_sold", "value": "{{ dbp_inc_gst }}" },
    { "key": "_wc_gtin", "value": "{{ bar_code }}" }
  ]
}
```

---

## Implementation Checklist

### Workflow Changes Required

1. **Single Product Mode** - Change LIMIT from 5 to 1 for testing
2. **WooCommerce Node** - Consider keeping HTTP Request for full meta_data support
3. **AI Title** - Implement Ollama + RAG for title optimisation
4. **Brand Lookup/Create** - Add logic to find/create yith_product_brand term
5. **Dimension Conversion** - Add Code node to convert mm → cm
6. **Warranty Formatting** - Format as "{months} Months"

### Pre-requisites

- Qdrant vector DB running for RAG title examples
- Ollama available for AI title generation
- WooCommerce brand taxonomy endpoint confirmed (`yith_product_brand`)

---

## Notes

- `stock_code` remains in `acf_supplier_product_id` for inventory sync workflows
- `manufacturer_sku` is used as WooCommerce SKU (customer-facing)
- Pricing workflow to be built separately - Publisher just uses RRP as regular_price
- Image optimisation (square cropping) deferred to separate workflow
