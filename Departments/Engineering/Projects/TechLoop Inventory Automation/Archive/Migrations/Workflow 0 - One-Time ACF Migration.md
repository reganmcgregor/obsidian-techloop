> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 0: One-Time ACF Migration

**Purpose:** Populate ACF supplier fields for all existing WooCommerce products
**Type:** One-time migration (run once, then disable)
**Trigger:** Manual execution only
**Priority:** CRITICAL - Must run before any sync workflows

---

## Overview

Currently, only 1 product in WooCommerce has ACF supplier fields populated. Before we can start automated syncing, we need to:

1. Match all WooCommerce products to Leader products
2. Populate ACF fields: `supplier_name`, `supplier_product_id`, `supplier_dbp`, `supplier_mpn`, `supplier_barcode`

**Matching Logic:** WooCommerce `sku` = Leader `manufacturer_sku`

**Assumption:** All products are from "Leader" supplier

---

## Workflow Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ WORKFLOW 0: One-Time ACF Migration                              │
└─────────────────────────────────────────────────────────────────┘

1. [Manual Trigger]
   │
2. [HTTP Request] Fetch Leader XML feed
   │
3. [Code] Parse XML to array
   │
4. [Code] Transform Leader data (create lookup by manufacturer_sku)
   │
5. [WooCommerce] List all products (paginated, 100 per page)
   │
6. [Code] Match WC products to Leader by SKU
   │
   ├─ Matched → Prepare ACF update
   │
   └─ Unmatched → Log to separate output
   │
7. [Code] Batch matched products (25 per batch)
   │
8. [Loop] For each batch:
   │   │
   │   ├─ [WooCommerce] Batch update products with ACF fields
   │   │
   │   ├─ [Code] Log success
   │   │
   │   └─ [Wait] 2 seconds (rate limiting)
   │
9. [Code] Generate migration report
   │
10. [Email] Send report (optional)
```

---

## Node-by-Node Specification

### Node 1: Manual Trigger
**Type:** Manual Trigger
**Settings:** None
**Purpose:** Manual execution only - this workflow should NEVER run on a schedule

---

### Node 2: Fetch Leader XML Feed
**Type:** HTTP Request
**Method:** GET
**URL:** `https://partner.leadersystems.com.au/WSDataFeed.asmx/DownLoad?CustomerCode=U2FsdGVkX19iL70lfWvWzLkZdA25pj1GfrS5R8itZoTxERYgWe23r%2BKRKDpCvblP&WithHeading=true&WithLongDescription=true&DataType=1`

**Settings:**
- Response Format: String (will be XML)
- Timeout: 60000ms (60 seconds)

**Output:** Raw XML string containing all Leader products

---

### Node 3: Parse Leader XML
**Type:** Code (JavaScript)
**Purpose:** Parse XML to JavaScript array

**Code:**
```javascript
const xml2js = require('xml2js');

// Input: $input.first().json (contains the XML string)
const xmlString = $input.first().json.data;

// Parse XML
const parser = new xml2js.Parser({ explicitArray: false });
const result = await parser.parseStringPromise(xmlString);

// Extract items from RSS feed
const items = result.rss.channel.item;

// Return array of items
return items.map(item => ({ json: item }));
```

**Output:** Array of Leader products (raw XML structure)

---

### Node 4: Transform Leader Data (Create Lookup Map)
**Type:** Code (JavaScript)
**Purpose:** Create a lookup object keyed by manufacturer_sku for fast matching

**Code:**
```javascript
// Copy the toSnakeCase function from n8n-utilities.js
function toSnakeCase(str) {
    if (!str || typeof str !== 'string') return str;
    return str
        .replace(/([A-Z]+)([A-Z][a-z])/g, '$1_$2')
        .replace(/([a-z\d])([A-Z])/g, '$1_$2')
        .replace(/[\s-]+/g, '_')
        .toLowerCase()
        .replace(/_+/g, '_')
        .replace(/^_|_$/g, '');
}

// Create lookup object: { manufacturer_sku: leader_data }
const leaderLookup = {};

for (const item of $input.all()) {
    const leaderProduct = item.json;

    // Extract key fields (handle XML array format)
    const manufacturerSku = Array.isArray(leaderProduct.ManufacturerSKU)
        ? leaderProduct.ManufacturerSKU[0]
        : leaderProduct.ManufacturerSKU;

    const stockCode = Array.isArray(leaderProduct.StockCode)
        ? leaderProduct.StockCode[0]
        : leaderProduct.StockCode;

    const dbp = Array.isArray(leaderProduct.DBP)
        ? leaderProduct.DBP[0]
        : leaderProduct.DBP;

    const barCode = Array.isArray(leaderProduct.BarCode)
        ? leaderProduct.BarCode[0]
        : leaderProduct.BarCode;

    // Store in lookup (key = manufacturer_sku for matching)
    if (manufacturerSku) {
        leaderLookup[manufacturerSku] = {
            stock_code: stockCode,
            manufacturer_sku: manufacturerSku,
            dbp: dbp,
            bar_code: barCode
        };
    }
}

// Return as single item with lookup object
return [{ json: { leaderLookup, totalLeaderProducts: Object.keys(leaderLookup).length } }];
```

**Output:** Single item containing `leaderLookup` object

---

### Node 5: Fetch All WooCommerce Products
**Type:** WooCommerce (MCP Tool)
**Operation:** List Products
**Function:** `mcp__woocommerce_mcp__woocommerce-products-list`

**Parameters:**
```json
{
  "per_page": 100,
  "status": "publish",
  "orderby": "id",
  "order": "asc"
}
```

**Note:** This will need to handle pagination. Use a loop to fetch all pages:
- Check response headers for `X-WP-Total` and `X-WP-TotalPages`
- Loop through all pages incrementing `page` parameter

**Alternative:** If using HTTP Request node directly:
```
GET https://www.techloop.com.au/wp-json/wc/v3/products?per_page=100&page=1&status=publish
Authorization: Basic [base64(consumer_key:consumer_secret)]
```

**Output:** Array of ALL WooCommerce products

---

### Node 6: Match WC Products to Leader
**Type:** Code (JavaScript)
**Purpose:** Match WooCommerce SKU to Leader manufacturer_sku and prepare updates

**Code:**
```javascript
// Get the leader lookup from previous node
const leaderLookup = $node["Transform Leader Data"].json.leaderLookup;

// Get all WC products
const wcProducts = $input.all();

const matched = [];
const unmatched = [];

for (const item of wcProducts) {
    const wcProduct = item.json;

    // Match by SKU (WC sku = Leader manufacturer_sku)
    const wcSku = wcProduct.sku;
    const leaderMatch = leaderLookup[wcSku];

    if (leaderMatch) {
        // MATCH FOUND - Prepare update payload
        matched.push({
            wc_id: wcProduct.id,
            sku: wcProduct.sku,
            name: wcProduct.name,
            // Current ACF values (may be null)
            current_supplier_name: wcProduct.meta_data?.find(m => m.key === 'supplier_name')?.value || null,
            current_supplier_product_id: wcProduct.meta_data?.find(m => m.key === 'supplier_product_id')?.value || null,
            // New ACF values from Leader
            leader_stock_code: leaderMatch.stock_code,
            leader_manufacturer_sku: leaderMatch.manufacturer_sku,
            leader_dbp: leaderMatch.dbp,
            leader_bar_code: leaderMatch.bar_code,
            // Update payload for WooCommerce API
            update_payload: {
                id: wcProduct.id,
                meta_data: [
                    { key: "supplier_name", value: "Leader" },
                    { key: "supplier_product_id", value: leaderMatch.stock_code },
                    { key: "supplier_dbp", value: leaderMatch.dbp },
                    { key: "supplier_mpn", value: leaderMatch.manufacturer_sku },
                    { key: "supplier_barcode", value: leaderMatch.bar_code }
                ]
            }
        });
    } else {
        // NO MATCH - Log for review
        unmatched.push({
            wc_id: wcProduct.id,
            sku: wcProduct.sku,
            name: wcProduct.name,
            reason: "No matching Leader product found (WC sku not in Leader manufacturer_sku)"
        });
    }
}

// Return both arrays for different branches
return [
    {
        json: {
            matched,
            unmatched,
            stats: {
                total_wc_products: wcProducts.length,
                matched_count: matched.length,
                unmatched_count: unmatched.length,
                match_rate: ((matched.length / wcProducts.length) * 100).toFixed(2) + '%'
            }
        }
    }
];
```

**Output:** Object with `matched` and `unmatched` arrays, plus `stats`

---

### Node 7: Batch Matched Products
**Type:** Code (JavaScript)
**Purpose:** Split matched products into batches of 25 for API rate limiting

**Code:**
```javascript
// Copy batchArray function from n8n-utilities.js
function batchArray(array, size) {
    if (!Array.isArray(array) || size <= 0) return [array];
    const batches = [];
    for (let i = 0; i < array.length; i += size) {
        batches.push(array.slice(i, i + size));
    }
    return batches;
}

const matched = $input.first().json.matched;
const batches = batchArray(matched, 25);

// Return each batch as separate item for loop
return batches.map((batch, index) => ({
    json: {
        batch_number: index + 1,
        batch_size: batch.length,
        products: batch
    }
}));
```

**Output:** Array of batches (each batch = 25 products max)

---

### Node 8: Loop Through Batches
**Type:** Loop Over Items
**Purpose:** Process each batch sequentially

---

### Node 8a: Update WooCommerce Products (Inside Loop)
**Type:** WooCommerce Batch Update OR HTTP Request
**Operation:** Batch Update Products

**Option A - Using WooCommerce Batch API (Recommended):**

**HTTP Request Node:**
```
POST https://www.techloop.com.au/wp-json/wc/v3/products/batch
Authorization: Basic [base64(consumer_key:consumer_secret)]
Content-Type: application/json
```

**Body:**
```javascript
{
  "update": $json.products.map(p => p.update_payload)
}
```

**Option B - Individual Updates (Slower):**
Loop through `$json.products` and call `mcp__woocommerce_mcp__woocommerce-products-update` for each

---

### Node 8b: Wait Between Batches
**Type:** Wait
**Duration:** 2000ms (2 seconds)
**Purpose:** Rate limiting to avoid WooCommerce API throttling

---

### Node 9: Generate Migration Report
**Type:** Code (JavaScript)
**Purpose:** Create summary report of migration

**Code:**
```javascript
const matchData = $node["Match WC Products to Leader"].json;
const stats = matchData.stats;
const unmatched = matchData.unmatched;

// Generate report
const report = {
    migration_date: new Date().toISOString(),
    summary: {
        total_wc_products: stats.total_wc_products,
        matched_and_updated: stats.matched_count,
        unmatched: stats.unmatched_count,
        match_rate: stats.match_rate
    },
    unmatched_products: unmatched.map(p => ({
        sku: p.sku,
        name: p.name,
        wc_id: p.wc_id,
        reason: p.reason
    })),
    recommendation: stats.unmatched_count > 0
        ? "Review unmatched products - they may need manual ACF field population or SKU correction"
        : "All products matched successfully! Ready to start sync workflows."
};

return [{ json: report }];
```

**Output:** Migration report object

---

### Node 10: Send Report Email (Optional)
**Type:** Send Email (SMTP) - Optional
**To:** Operations email
**Subject:** `TechLoop ACF Migration Complete - ${stats.matched_count} products updated`

**Body Template:**
```html
<h2>ACF Supplier Fields Migration Report</h2>

<h3>Summary</h3>
<ul>
    <li><strong>Total WooCommerce Products:</strong> {{stats.total_wc_products}}</li>
    <li><strong>Successfully Matched & Updated:</strong> {{stats.matched_count}}</li>
    <li><strong>Unmatched:</strong> {{stats.unmatched_count}}</li>
    <li><strong>Match Rate:</strong> {{stats.match_rate}}</li>
</ul>

<h3>ACF Fields Populated:</h3>
<ul>
    <li>supplier_name = "Leader"</li>
    <li>supplier_product_id = Leader StockCode</li>
    <li>supplier_dbp = Leader DBP</li>
    <li>supplier_mpn = Leader ManufacturerSKU</li>
    <li>supplier_barcode = Leader BarCode</li>
</ul>

{{#if stats.unmatched_count}}
<h3>⚠️ Unmatched Products</h3>
<p>The following products could not be matched to Leader's catalog:</p>
<ul>
{{#each unmatched_products}}
    <li>[{{this.sku}}] {{this.name}} (ID: {{this.wc_id}})</li>
{{/each}}
</ul>
<p><strong>Action Required:</strong> Review these products - they may need manual ACF field population or SKU corrections.</p>
{{/if}}

<hr>
<p><strong>Next Steps:</strong> {{recommendation}}</p>
```

---

## Pre-Flight Checklist

Before running this migration:

- [ ] Backup WooCommerce database
- [ ] Test on staging environment first (if available)
- [ ] Verify Leader XML feed is accessible
- [ ] Verify WooCommerce API credentials work
- [ ] Review expected match count (approximately how many products do you expect?)

---

## Expected Results

Based on typical WooCommerce + Leader setup:

**Best Case:**
- Match rate: 90-100%
- All products successfully updated with ACF fields
- Ready to start sync workflows immediately

**Realistic Case:**
- Match rate: 70-90%
- Some products need manual SKU correction or manual ACF population
- Most products ready to sync

**Potential Issues:**

1. **SKU Mismatch:** WooCommerce SKU doesn't match Leader ManufacturerSKU
   - **Solution:** Manually correct SKUs or populate ACF fields manually

2. **Products from Other Suppliers:** Some products not in Leader's catalog
   - **Solution:** These will remain unmatched (expected)

3. **Duplicate SKUs:** Multiple WC products with same SKU
   - **Solution:** Fix duplicate SKUs first

---

## Verification Query

After running migration, verify results with this Supabase query:

```sql
-- Once you run Workflow 2 (WC Mirror), check ACF population
SELECT
    COUNT(*) as total_products,
    COUNT(acf_supplier_product_id) as with_supplier_id,
    COUNT(*) - COUNT(acf_supplier_product_id) as missing_supplier_id,
    ROUND(
        (COUNT(acf_supplier_product_id)::numeric / COUNT(*) * 100),
        2
    ) as population_rate
FROM tl_wc_products_mirror
WHERE status = 'publish';

-- Should show ~80-100% population rate
```

---

## Alternative Approach: Script-Based Migration

If you prefer to run this as a standalone script instead of n8n workflow, here's a Node.js script:

**File:** `migration-populate-acf.js`

```javascript
const axios = require('axios');
const xml2js = require('xml2js');

// Configuration
const WC_URL = 'https://www.techloop.com.au';
const WC_KEY = 'YOUR_CONSUMER_KEY';
const WC_SECRET = 'YOUR_CONSUMER_SECRET';
const LEADER_FEED_URL = 'https://partner.leadersystems.com.au/...';

async function main() {
    console.log('Starting ACF Migration...');

    // 1. Fetch Leader feed
    console.log('Fetching Leader feed...');
    const leaderXml = await axios.get(LEADER_FEED_URL);
    const parser = new xml2js.Parser({ explicitArray: false });
    const leaderData = await parser.parseStringPromise(leaderXml.data);
    const leaderItems = leaderData.rss.channel.item;

    // 2. Create lookup by manufacturer_sku
    const leaderLookup = {};
    for (const item of leaderItems) {
        const sku = item.ManufacturerSKU;
        leaderLookup[sku] = {
            stock_code: item.StockCode,
            dbp: item.DBP,
            bar_code: item.BarCode
        };
    }

    console.log(`Leader products indexed: ${Object.keys(leaderLookup).length}`);

    // 3. Fetch WooCommerce products (paginated)
    let page = 1;
    let hasMore = true;
    const allProducts = [];

    while (hasMore) {
        const response = await axios.get(`${WC_URL}/wp-json/wc/v3/products`, {
            auth: { username: WC_KEY, password: WC_SECRET },
            params: { per_page: 100, page, status: 'publish' }
        });

        allProducts.push(...response.data);
        hasMore = response.data.length === 100;
        page++;
    }

    console.log(`WooCommerce products fetched: ${allProducts.length}`);

    // 4. Match and prepare updates
    const updates = [];
    let matched = 0;
    let unmatched = 0;

    for (const product of allProducts) {
        const match = leaderLookup[product.sku];

        if (match) {
            matched++;
            updates.push({
                id: product.id,
                meta_data: [
                    { key: 'supplier_name', value: 'Leader' },
                    { key: 'supplier_product_id', value: match.stock_code },
                    { key: 'supplier_dbp', value: match.dbp },
                    { key: 'supplier_mpn', value: product.sku },
                    { key: 'supplier_barcode', value: match.bar_code }
                ]
            });
        } else {
            unmatched++;
            console.log(`No match for: ${product.sku} - ${product.name}`);
        }
    }

    console.log(`
Matched: ${matched}, Unmatched: ${unmatched}`);
    console.log(`Match rate: ${((matched / allProducts.length) * 100).toFixed(2)}%`);

    // 5. Batch update (25 at a time)
    console.log('
Updating WooCommerce products...');
    for (let i = 0; i < updates.length; i += 25) {
        const batch = updates.slice(i, i + 25);

        await axios.post(`${WC_URL}/wp-json/wc/v3/products/batch`, {
            update: batch
        }, {
            auth: { username: WC_KEY, password: WC_SECRET }
        });

        console.log(`Updated batch ${Math.floor(i / 25) + 1}/${Math.ceil(updates.length / 25)}`);

        // Rate limiting
        await new Promise(resolve => setTimeout(resolve, 2000));
    }

    console.log('
Migration complete!');
}

main().catch(console.error);
```

---

## Post-Migration

Once migration is complete:

1. ✅ Verify ACF fields populated in WooCommerce admin
2. ✅ Run Workflow 2 (WC Mirror) to populate `tl_wc_products_mirror`
3. ✅ Check Supabase to verify `acf_supplier_product_id` populated
4. ✅ Ready to start Phase 2 workflows!

---

## Notes

- **One-time only:** This workflow should be disabled after successful execution
- **Idempotent:** Safe to re-run if needed (will overwrite existing ACF values)
- **Backup recommended:** Always backup before mass updates
- **Test first:** Run on staging/sample products if possible
