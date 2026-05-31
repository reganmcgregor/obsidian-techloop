# Phase 10 — TL_Product_Publisher_Shopify rewrite spec (working scratch)

Clone is live + inactive: **`pgWWBz9f6RXLXEIe`** (verbatim 55-node copy of frozen `xBSqJjECViFMgUNI`).
This file is the exact edit list to convert the WC clone → Shopify. Mechanical edits are Sonnet-delegatable;
the `productSet` Code node + GraphQL configs below are the design (apply verbatim).

Store facts: host `techloop-7.myshopify.com`, API `2025-07`, auth Custom Auth cred `4Kf8ZBknzZl6bvvi`,
NSW location GID `gid://shopify/Location/82672910534`. Postgres cred `BSoGuZ9BOv4OWqXf`,
Slack cred `DvtYgxYgI1p5Q3QX`, Ollama `mLc8L4ENefXAw6xJ`, Qdrant `XUbKGzKPJwEN8dVU`.

## Open dependency (flag to user)
`tl_shopify_collections_mirror` is **empty (0 rows)** — so there are NO category-collection GIDs to map to yet.
Brand auto-files fine (vendor smart collections key on `vendor EQUALS <brand>`). For category collections,
we don't yet know whether they're smart (rule-based on tag/product_type → auto-file) or manual (need
collectionAddProducts with a GID). **Decision for v1:** set `vendor` (brand auto-files), set `productType`
+ brand/category `tags` from the mapped names, and DEFER explicit category-collection joins until the
collections mirror is populated (Mirror tier extension or a one-off backfill). Note in spec doc as follow-up.

## Node disposition (55 nodes)

### KEEP AS-IS (no change)
Schedule Every 5 Minutes, Capture Start Time, Has Approved?, Still In Stock?, all AI/Qdrant nodes
(AI Optimise Title, Parse AI Title, AI Clean Description, Parse Clean Description, AI Short Description,
Parse Short Description, Search Similar Titles/Short Descs, Format Title/Short Desc Prompt, Embeddings
Ollama ×3, Prepare Title Document, Upsert Title to Qdrant, Default Data Loader), full image-processing
chain (Download Image, Has Image?, Get Image Info, Calculate Padding, Make Square, Add Padding, Cap Size,
Merge Image Result), Calculate Price, Build Summary, Published Any?, No Products, No Email, Mark Out of Stock,
Format Slack Blocks, Send Slack, Build Publish Blocks, Send Publish Message.

### MODIFY (Postgres queries / Code field names)
- **Fetch Approved Products** (id 3): keep query (status='approved' … LIMIT 1). No change needed.
- **Lock Product** (id 5): keep (UPDATE … status='processing' … RETURNING *). No change.
- **Get Fresh Leader Data** (id 6): keep (SELECT … tl_feed_leader_raw WHERE stock_code). No change.
- **Lookup Brand** (id 23): keep — still resolves a brand name string from tl_brand_map. We only need
  `wc_brand_name` (→ used as Shopify vendor string). No schema change.
- **Merge Brand Data** (id 24): keep — outputs `brand_name`. (brand_term_id now unused downstream.)
- **Mark Published** (id 12): REWRITE query →
  `UPDATE tl_onboarding_queue SET status='published', published_at=NOW(),
   shopify_product_gid='{gid}', shopify_product_id={numericId}, shopify_product_url='{url}',
   shopify_variant_gid='{variantGid}', shopify_inventory_item_gid='{invItemGid}',
   processing_started_at=NULL WHERE stock_code='{stock_code}';`
  Source the gid/id/url from the productSet response (see Parse productSet Result below).
- **Mark Failed** (id 14): REWRITE to read Shopify userErrors instead of WC `$json.message`
  (parse from Parse productSet Result `error_message`).
- **Create Optimization Record** (id 13): change `wc_product_id` value to the Shopify numeric id
  (keep column name for now — it's the PK conflict target; tl_product_optimizations stays WC-keyed,
  joined by stock_code/sku). Keep ON CONFLICT.
- **Log Execution** (id 29): change workflow_name → 'TL_Product_Publisher_Shopify'.

### REPLACE (remove / swap nodes)
- **Has Brand ID?** (id 28) + **Create Brand If Missing** (id 25) + **Resolve Brand ID** (id 26):
  REMOVE all three (no brand-create on Shopify — vendor is free-text). Rewire Merge Brand Data → Download Image.
- **Build WC Product Body** (id 27): REPLACE jsCode with "Build productSet Input" (below). Rename node.
- **Upload to WC Media** (4cd1eb89…) + **Update Media Metadata** (update-media-meta): REPLACE with the
  staged-upload chain (Get Token → stagedUploadsCreate → HTTP multipart upload → capture resourceUrl).
  Feeds Merge Image Result with `staged_resource_url` instead of `wc_media_id/url`.
- **Create WooCommerce Product** (id 10): REPLACE with two nodes — `Get Shopify Token` (HTTP) →
  `Shopify productSet` (HTTP GraphQL). Rename.
- **Handle Result** (id 11): keep IF but switch condition to `{{ $json.product_id }}` exists/gt (from
  Parse productSet Result), driven by a new "Parse productSet Result" Code node between productSet and Handle Result.

## Auth + GraphQL node configs (apply verbatim)

### Get Shopify Token (HTTP Request, typeVersion 4.2)
```json
{ "method": "POST",
  "url": "https://techloop-7.myshopify.com/admin/oauth/access_token",
  "authentication": "genericCredentialType", "genericAuthType": "httpCustomAuth",
  "sendBody": true, "contentType": "form-urlencoded", "specifyBody": "keypair",
  "bodyParameters": { "parameters": [ { "name": "grant_type", "value": "client_credentials" } ] },
  "options": {} }
// credentials: { "httpCustomAuth": { "id": "4Kf8ZBknzZl6bvvi", "name": "Shopify Client Credentials" } }
```

### Build productSet Input (Code node, replaces Build WC Product Body)
Pulls from Merge Image Result (image), Calculate Price, Merge Brand Data, Get Fresh Leader Data, Parse AI Title,
Parse Clean Description, Parse Short Description. Emits { gql, vars, stock_code }.
```js
const price = $('Calculate Price').first().json;
const leader = $('Get Fresh Leader Data').first().json;
const brand = $('Merge Brand Data').first().json;
const title = $('Parse AI Title').first().json.optimised_title;
const cleanDesc = $('Parse Clean Description').first().json.clean_description || leader.product_name_2 || '';
const shortDesc = ($('Parse Short Description').first() || {}).json?.short_description || '';
const img = $input.first().json; // Merge Image Result → staged_resource_url

const NSW = 'gid://shopify/Location/82672910534';
const sku = leader.manufacturer_sku || leader.stock_code;
const qty = parseInt(leader.availability_total || 0, 10);
const weightVal = parseFloat(price.weight_kg || leader.weight || 0) || 0;

// metafields — only TRUE custom fields (GTIN→barcode, COGS→inventoryItem.cost are native)
const metafields = [
  { namespace:'custom', key:'short_description', type:'multi_line_text_field', value: shortDesc || '' },
  { namespace:'custom', key:'supplier_status',  type:'single_line_text_field', value:'active' },
  { namespace:'custom', key:'supplier_cost',    type:'number_decimal', value:String(leader.dbp_inc_gst || 0) },
].filter(m => m.value !== '' && m.value != null);
if (price.warranty_formatted) metafields.push({ namespace:'custom', key:'warranty', type:'single_line_text_field', value:String(price.warranty_formatted) });
if (leader.manufacturer_sku)  metafields.push({ namespace:'custom', key:'model_number', type:'single_line_text_field', value:String(leader.manufacturer_sku) });

const variant = {
  sku: sku,
  price: String(price.regular_price),
  barcode: leader.bar_code || null,
  optionValues: [{ optionName: 'Title', name: 'Default Title' }],
  inventoryItem: { cost: String(price.cost_price || leader.dbp_inc_gst || 0), tracked: true,
    measurement: { weight: { value: weightVal, unit: 'KILOGRAMS' } } },
  inventoryQuantities: [{ locationId: NSW, name: 'available', quantity: qty }],
};

const input = {
  title: title,
  descriptionHtml: cleanDesc,
  vendor: brand.brand_name || leader.manufacturer || '',
  productType: leader.mapped_wc_category_name || $('Lock Product').first().json.mapped_wc_category_name || '',
  status: 'DRAFT',
  tags: [brand.brand_name, leader.category_name].filter(Boolean),
  productOptions: [{ name: 'Title', values: [{ name: 'Default Title' }] }],
  variants: [variant],
  metafields: metafields,
};

// media: only attach when we have a processed staged upload URL
const media = img.staged_resource_url
  ? [{ mediaContentType: 'IMAGE', originalSource: img.staged_resource_url, alt: title }]
  : (leader.image_url ? [{ mediaContentType:'IMAGE', originalSource: leader.image_url, alt: title }] : []);
if (media.length) input.media = undefined; // media is a separate mutation arg, not in input

// Spaced braces so n8n parser is happy; productSet input + media as separate vars
const gql = 'mutation Set($input: ProductSetInput!, $media: [CreateMediaInput!]) { '
  + 'productSet(synchronous: true, input: $input, media: $media) { '
  + 'product { id legacyResourceId handle onlineStorePreviewUrl '
  + 'variants(first: 1) { nodes { id sku inventoryItem { id } } } } '
  + 'userErrors { field message } } }';

return [{ json: { gql, vars: { input, media }, stock_code: leader.stock_code } }];
```

### Shopify productSet (HTTP Request GraphQL, typeVersion 4.2)
```json
{ "method": "POST",
  "url": "https://techloop-7.myshopify.com/admin/api/2025-07/graphql.json",
  "sendHeaders": true,
  "headerParameters": { "parameters": [ { "name": "X-Shopify-Access-Token",
    "value": "={{ $('Get Shopify Token').first().json.access_token }}" } ] },
  "sendBody": true, "contentType": "json", "specifyBody": "keypair",
  "bodyParameters": { "parameters": [
    { "name": "query",     "value": "={{ $json.gql }}" },
    { "name": "variables", "value": "={{ $json.vars }}" } ] },
  "options": { "response": { "response": { "neverError": true } } } }
```

### Parse productSet Result (Code node, NEW — between productSet and Handle Result)
```js
const r = $input.first().json;
const ps = r?.data?.productSet;
const errs = (ps?.userErrors || []).concat(r?.errors || []);
const product = ps?.product;
const ok = !!product?.id && errs.length === 0;
const variant = product?.variants?.nodes?.[0];
return [{ json: {
  ok,
  product_gid: product?.id || null,
  product_id: product?.legacyResourceId ? Number(product.legacyResourceId) : null,
  product_url: product?.onlineStorePreviewUrl || (product?.handle ? 'https://www.techloop.com.au/products/' + product.handle : null),
  variant_gid: variant?.id || null,
  inventory_item_gid: variant?.inventoryItem?.id || null,
  error_message: errs.map(e => e.message).join('; ').slice(0,500) || null,
  stock_code: $('Build productSet Input').first().json.stock_code,
} }];
```
Handle Result condition → `{{ $json.ok }}` is true.

### Staged image upload chain (replaces Upload to WC Media + Update Media Metadata)
Sits after Cap Size (which outputs processed binary on `data`). Three nodes:
1. **stagedUploadsCreate** (HTTP GraphQL, same auth header pattern). Code before it builds vars:
   `{ input:[{ filename: slug+'.jpg', mimeType:'image/jpeg', httpMethod:'POST', resource:'PRODUCT_IMAGE', fileSize:String(bytes) }] }`
   mutation: `mutation u($input:[StagedUploadInput!]!){ stagedUploadsCreate(input:$input){ stagedTargets{ url resourceUrl parameters{ name value } } userErrors{ field message } } }`
2. **HTTP multipart POST** to `stagedTargets[0].url` — append each `parameters[]` name/value then the binary `file` field. (n8n HTTP multipart-form-data; binary from Cap Size `data`.)
3. **Code: capture** → `{ staged_resource_url: stagedTargets[0].resourceUrl }` merged into Merge Image Result path.
NOTE: if multipart from binary proves fiddly in n8n, fallback = sideload `leader.image_url` directly
(skip staging) — already handled in Build productSet Input media fallback. Try staged first per user decision (keep full pipeline).

## Build / test procedure (when MCP back)
1. Apply MODIFY/REPLACE edits via n8n_update_partial_workflow (incremental addNode/removeNode/updateNode).
2. n8n_validate_workflow until clean (ignore pre-existing Slack `operation` advisory).
3. Hand-approve ONE good queue row: `UPDATE tl_onboarding_queue SET status='approved' WHERE stock_code='<pick a clean mapped row>'`
   (pick one with image_url, mapped brand, in stock — e.g. an LG/Samsung monitor from the 204-row view).
4. Temp webhook → run once → verify on live store: draft product, variant SKU/price/barcode/cost/inventory@NSW,
   custom metafields, image attached, vendor → brand smart collection membership.
5. Verify tl_onboarding_queue got shopify_* ids; tl_workflow_executions logged; Slack publish card posted.
6. Remove temp webhook, set schedule `*/5 * * * *`, deactivate→reactivate.
```
