> [!info] Part of [[Master Note - Inventory Automation]]

# Code Updates - Fix Space Bug & Add Unmatched Export

**Date:** 2026-01-13
**Status:** Ready to implement

## Issues to Fix

1. **"Leader " with trailing space** - Fix supplier_name field
2. **Export unmatched products** - Save list to markdown for manual review
3. **Improved SKU matching** - Normalize SKUs to catch variations (spaces, dashes, case)

## Changes Required

### 1. Update "Prepare Single Update" Node

**Current code has:** `value: 'Leader'` (but somehow stores `'Leader '`)

**Replace entire code with:**

```javascript
// Get the matched product data
const product = $input.item.json;

// Format meta_data array for WooCommerce - ensure no trailing spaces
const meta_data = [
  { key: 'supplier_name', value: 'Leader' },  // No trailing space!
  { key: 'supplier_product_id', value: product.leader_stock_code },
  { key: 'supplier_dbp', value: product.leader_dbp },
  { key: 'supplier_mpn', value: product.leader_manufacturer_sku },
  { key: 'supplier_barcode', value: product.leader_barcode }
];

console.log(`Preparing update for product ${product.wc_id}: ${product.wc_sku}`);

return {
  json: {
    product_id: product.wc_id,
    meta_data: meta_data,
    sku: product.wc_sku,
    name: product.wc_name
  }
};
```

### 2. Update "Match WC to Leader" Node

**Replace entire code with:**

```javascript
// Get the Leader lookup and WC products
const leaderLookup = $('Create Leader Lookup').first().json.leaderLookup;
const wcProducts = $input.all();

// Helper function to normalize SKU for matching
function normalizeSKU(sku) {
  if (!sku) return '';
  return String(sku)
    .toUpperCase()
    .replace(/[\s-_]/g, '')  // Remove spaces, dashes, underscores
    .trim();
}

// Create normalized lookup
const normalizedLookup = {};
for (const [originalSku, data] of Object.entries(leaderLookup)) {
  const normalizedSku = normalizeSKU(originalSku);
  normalizedLookup[normalizedSku] = {
    ...data,
    original_sku: originalSku
  };
}

const matched = [];
const unmatched = [];

for (const item of wcProducts) {
  const wcProduct = item.json;
  const wcSku = wcProduct.sku;
  const normalizedWcSku = normalizeSKU(wcSku);

  // Try to find match in Leader feed (normalized)
  if (normalizedWcSku && normalizedLookup[normalizedWcSku]) {
    const leaderData = normalizedLookup[normalizedWcSku];

    matched.push({
      wc_id: wcProduct.id,
      wc_sku: wcSku,
      wc_name: wcProduct.name,
      leader_stock_code: leaderData.stock_code,
      leader_manufacturer_sku: leaderData.manufacturer_sku,
      leader_dbp: leaderData.dbp,
      leader_barcode: leaderData.bar_code
    });
  } else {
    unmatched.push({
      wc_id: wcProduct.id,
      wc_sku: wcSku || 'NO_SKU',
      wc_name: wcProduct.name,
      reason: wcSku ? 'SKU not found in Leader feed' : 'Product has no SKU'
    });
  }
}

console.log(`Matched: ${matched.length}, Unmatched: ${unmatched.length}`);

// Store both matched and unmatched in workflow static data
const staticData = this.getWorkflowStaticData('global');
staticData.matchedProducts = matched;
staticData.unmatchedProducts = unmatched;

// Return matched products as separate items for looping
return matched.map(product => ({ json: product }));
```

### 3. Update "Generate Report" Node

**Replace entire code with:**

```javascript
// Get updated products from loop
const allItems = $input.all();

// Get matched and unmatched products from workflow static data
const staticData = this.getWorkflowStaticData('global');
const unmatchedProducts = staticData.unmatchedProducts || [];
const matchedProducts = staticData.matchedProducts || [];

// Count successful updates
let successful = allItems.length;

const timestamp = new Date().toISOString();
const dateOnly = timestamp.split('T')[0];

// Create markdown content for report
let markdownContent = `# Leader Migration Report - ${dateOnly}

`;
markdownContent += `**Generated:** ${timestamp}

`;
markdownContent += `## Summary

`;
markdownContent += `- **Total Products in WooCommerce:** ${successful + unmatchedProducts.length}
`;
markdownContent += `- **Successfully Matched & Updated:** ${successful}
`;
markdownContent += `- **Unmatched Products:** ${unmatchedProducts.length}
`;
markdownContent += `- **Success Rate:** ${((successful / (successful + unmatchedProducts.length)) * 100).toFixed(2)}%

`;

// Matched Products Section
markdownContent += `## ✅ Matched Products (${successful})

`;
markdownContent += `These products were successfully matched to Leader's feed and ACF fields were populated.

`;

if (matchedProducts.length > 0) {
  markdownContent += `| WC ID | WC SKU | Product Name | Leader Stock Code | Leader MPN |
`;
  markdownContent += `|-------|--------|--------------|-------------------|------------|
`;

  for (const product of matchedProducts) {
    const name = product.wc_name.replace(/\|/g, '\|'); // Escape pipes
    markdownContent += `| ${product.wc_id} | \`${product.wc_sku}\` | ${name} | \`${product.leader_stock_code}\` | \`${product.leader_manufacturer_sku}\` |
`;
  }
} else {
  markdownContent += `*No matched products.*
`;
}

// Unmatched Products Section
markdownContent += `
## ⚠️ Unmatched Products (${unmatchedProducts.length})

`;
markdownContent += `These products need manual review. They either have no SKU or their SKU doesn't match any product in the Leader feed.

`;

if (unmatchedProducts.length > 0) {
  markdownContent += `| WC ID | SKU | Product Name | Reason |
`;
  markdownContent += `|-------|-----|--------------|--------|
`;

  for (const product of unmatchedProducts) {
    const sku = product.wc_sku || 'NO_SKU';
    const name = product.wc_name.replace(/\|/g, '\|');
    const reason = product.reason;
    markdownContent += `| ${product.wc_id} | \`${sku}\` | ${name} | ${reason} |
`;
  }
} else {
  markdownContent += `*No unmatched products found.*
`;
}

markdownContent += `
## Next Steps

`;
markdownContent += `### For Matched Products
`;
markdownContent += `- ✅ ACF fields populated with Leader supplier data
`;
markdownContent += `- ✅ No further action needed

`;
markdownContent += `### For Unmatched Products
`;
markdownContent += `1. Review each unmatched product in WooCommerce
`;
markdownContent += `2. Check if the SKU exists in Leader's feed with a different format
`;
markdownContent += `3. Update WooCommerce SKUs to match Leader if needed
`;
markdownContent += `4. For products confirmed to be from Leader, set supplier to "Leader" manually
`;
markdownContent += `5. For products NOT from Leader, update supplier metadata to correct supplier
`;

// Create report object
const report = {
  migration_complete: true,
  timestamp: timestamp,
  summary: {
    total_products: successful + unmatchedProducts.length,
    successfully_updated: successful,
    unmatched: unmatchedProducts.length,
    success_rate: `${((successful / (successful + unmatchedProducts.length)) * 100).toFixed(2)}%`
  },
  markdown_content: markdownContent,
  markdown_filename: `Leader Migration Report - ${dateOnly}.md`,
  matched_count: successful,
  unmatched_count: unmatchedProducts.length
};

console.log(`Migration Complete: ${successful} matched, ${unmatchedProducts.length} unmatched`);

return { json: report };
```

### 4. No Additional Node Needed!

The "Generate Report" node will output the markdown content in the `markdown_content` field. You can manually copy it to create a file in Obsidian.

## Implementation Steps

**STATUS: ✅ COMPLETE - All code updates applied via API**

1. ✅ Open workflow in n8n (https://n8n.reganmcgregor.com.au/workflow/ABT5WhqLikVLvbBc)
2. ✅ Updated "Prepare Single Update" node code
3. ✅ Updated "Match WC to Leader" node code
4. ✅ Updated "Generate Report" node code
5. ✅ Fixed "Update WooCommerce Product" node (changed operation from "create" to "update")
6. ✅ Removed Wait node and reconnected loop
7. ✅ Added "Send Email Report" node
8. **NEXT:** Configure SMTP credentials for email node (if not done)
9. **NEXT:** Run workflow (Execute Workflow button)
10. **NEXT:** Check your email (me@reganmcgregor.com.au) for the migration report

## Expected Results

After running the workflow:

- **More products will match** (due to normalized SKU matching - expect significantly more than 112)
- All matched products will have ACF fields populated with **"Leader"** (no trailing space)
- Console will show: `Matched: X, Unmatched: Y`
- Generate Report output will contain complete markdown with:
  - Summary statistics
  - Table of all matched products
  - Table of all unmatched products
  - Next steps guidance

## Next Phase

After review of unmatched products, create a second workflow to:
- Set `supplier_name` to "Leader" for all unmatched products
- Leave other supplier fields empty (they're not in Leader's feed)
