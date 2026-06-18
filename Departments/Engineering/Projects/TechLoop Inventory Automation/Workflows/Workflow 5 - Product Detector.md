# Workflow 5: Product Detector (TL_Product_Detector)

**Status:** Deployed
**n8n Name:** `TL_Product_Detector`
**n8n ID:** `rnUjGGsdD1jdaFIg`
**Direct Link:** [Open in n8n](https://workflows.labgregor.dev/workflow/rnUjGGsdD1jdaFIg)
**Schedule:** Every 2 hours at :45 (after Leader Ingest at :00)
**Cron:** `45 */2 * * *`

---

## Purpose

The Product Detector identifies **new products** in the Leader feed that do not exist in WooCommerce and queues them for human review. This is the first stage of the product onboarding pipeline.

---

## Prerequisites

- [[Workflow 1 - Leader Ingest]] must run first (populates `tl_feed_leader_raw`)
- [[Workflow 2 - WooCommerce Mirroring]] should be up-to-date (populates `tl_wc_products_mirror`)
- Phase 6 database schema deployed ([[Database Schema - Product Automation]])

---

## Workflow Logic

### Detection Criteria

A product is eligible for onboarding when:
1. It appears in the Leader feed (seen within last 4 hours)
2. It has stock (`availability_total > 0`)
3. It has a price (`dbp_inc_gst > 0`)
4. It does NOT exist in WooCommerce (no matching `acf_supplier_product_id`)
5. It is NOT already in the onboarding queue
6. Its category is NOT blocked in `tl_category_map`

### Status Assignment

| Condition | Status Set |
|-----------|------------|
| Category has `is_blocked = true` | `blocked` |
| Category has `auto_approve = true` | `approved` |
| Otherwise | `pending_review` |

---

## Node Architecture

| # | Node Type | Node Name | Purpose |
|---|-----------|-----------|---------|
| 1 | Schedule Trigger | Schedule Every 2 Hours at :45 | Cron: `45 */2 * * *` |
| 2 | Code | Capture Start Time | Records workflow start timestamp |
| 3 | Postgres | Fetch New Products | Query `vw_new_products_for_onboarding LIMIT 100` |
| 4 | IF | Has Products? | Check if any new products found |
| 5 | Code | Prepare Queue Data | Transform data for queue insert |
| 6 | Postgres | Insert to Queue | INSERT into `tl_onboarding_queue` |
| 7 | Code | Build Summary | Aggregate counts by status/category/brand |
| 8 | Supabase | Log Execution | INSERT into `tl_workflow_executions` |
| 9 | IF | Products Detected? | Check if any products were queued |
| 10 | Code | Format Slack Blocks | Build summary notification |
| 11 | Slack | Send Slack | Post summary to #product-detector |
| 12 | Filter | Filter Pending Review | Get only products needing review |
| 13 | Code | Build HITL Blocks | Create interactive review messages |
| 14 | Slack | Send HITL Message | Post review requests to #product-detector |
| 15 | No Operation | No Products Found | End when no products found |
| 16 | No Operation | No Slack Needed | End when no products detected |

---

## Node Details

### Node 1: Schedule Trigger

```
Trigger: Cron
Expression: 45 */2 * * *
```

Runs 45 minutes after each even hour, ensuring Leader Ingest has completed.

---

### Node 2: Fetch New Products (Supabase)

**Operation:** Select
**Table:** (Use RPC or raw SQL)

```sql
SELECT * FROM vw_new_products_for_onboarding
ORDER BY availability_total DESC, dbp_inc_gst DESC
LIMIT 100;
```

**Why LIMIT 100?**
- Prevents overwhelming the queue with thousands of products at once
- Allows gradual rollout and human review
- Can be increased once process is validated

---

### Node 3: Has Products? (IF)

**Condition:** `{{ $json.length > 0 }}`

Routes to processing nodes if products found, otherwise skips to "No Products" end.

---

### Node 4: Batch 25 (Split In Batches)

**Batch Size:** 25

Processes products in batches to avoid timeouts and memory issues.

---

### Node 5: Prepare Insert Data (Code)

```javascript
// Transform each product for queue insertion
return items.map(item => {
  const data = item.json;

  // Determine initial status based on category mapping
  let status = 'pending_review';
  if (data.is_blocked === true) {
    status = 'blocked';
  } else if (data.auto_approve === true) {
    status = 'approved';
  }

  // Calculate priority (higher stock + higher value = higher priority)
  const stockScore = Math.min(data.availability_total || 0, 100);
  const priceScore = Math.min((data.dbp_inc_gst || 0) / 100, 50);
  const priority = Math.round(stockScore * 0.5 + priceScore);

  return {
    json: {
      stock_code: data.stock_code,
      product_name: data.product_name,
      product_name_2: data.product_name_2,
      manufacturer: data.manufacturer,
      manufacturer_sku: data.manufacturer_sku,
      bar_code: data.bar_code,
      category_code: data.category_code,
      category_name: data.category_name,
      subcategory_name: data.subcategory_name,
      dbp_ex_gst: data.dbp_ex_gst,
      dbp_inc_gst: data.dbp_inc_gst,
      rrp: data.rrp,
      image_url: data.image_url,
      availability_total: data.availability_total,
      weight: data.weight,
      length: data.length,
      width: data.width,
      height: data.height,
      warranty_length: data.warranty_length,
      mapped_wc_category_id: data.mapped_wc_category_id,
      mapped_wc_category_name: data.mapped_wc_category_name,
      status: status,
      priority: priority,
      raw_data: data  // Store full original data as JSONB
    }
  };
});
```

---

### Node 6: Insert to Queue (Supabase)

**Operation:** Insert
**Table:** `tl_onboarding_queue`

**On Conflict:** `stock_code` - DO NOTHING (idempotent)

This ensures re-running the workflow won't create duplicates.

---

### Node 7: Merge Results

**Mode:** Merge By Index

Combines all batch results for summary.

---

### Node 8: Build Summary (Code)

```javascript
// Aggregate results by status and category
const results = items.map(i => i.json);

const summary = {
  total: results.length,
  by_status: {},
  by_category: {},
  timestamp: new Date().toISOString()
};

results.forEach(r => {
  // Count by status
  summary.by_status[r.status] = (summary.by_status[r.status] || 0) + 1;

  // Count by category
  const cat = r.category_name || 'Unknown';
  summary.by_category[cat] = (summary.by_category[cat] || 0) + 1;
});

return [{ json: summary }];
```

---

### Node 9: Log Execution (Supabase)

**Operation:** Insert
**Table:** `tl_workflow_executions`

```json
{
  "workflow_name": "TL_Product_Detector",
  "execution_id": "{{ $execution.id }}",
  "status": "success",
  "records_processed": "{{ $json.total }}",
  "metadata": {
    "by_status": "{{ $json.by_status }}",
    "by_category": "{{ $json.by_category }}"
  }
}
```

---

### Node 9: Products Detected? (IF)

**Condition:** `{{ $('Build Summary').first().json.total_queued > 0 }}`

Only sends notifications if products were actually queued.

---

### Node 10: Format Slack Blocks (Code)

Builds a summary notification with:
- Total queued, failed, and duration
- Breakdown by status
- Top 10 brands and categories
- Execution ID and timestamp

---

### Node 11: Send Slack (Summary)

**Channel:** `#product-detector`
**Message Type:** Block Kit

Posts a summary card showing detection results.

---

### Node 12: Filter Pending Review

**Condition:** `status = 'pending_review'`

Filters queue items that need human review, passing them to HITL messaging.

---

### Node 13: Build HITL Blocks (Code)

Creates interactive Slack messages for human review with:
- Product name, stock code, MPN
- Brand, category, cost, stock level
- Product image (if available)
- **Three action buttons:**

**Buttons:**
1. **Approve** (Primary/Blue) - `action_id: approve_product`
2. **Reject** (Danger/Red) - `action_id: reject_product`
3. **🔍 Google Search** (Link) - Opens Google search with MPN or product name

**Google Search Feature:**
- Uses manufacturer SKU for precise results
- Falls back to product name if no SKU
- Opens in browser (perfect for mobile review)
- Helps research product specs, images, and pricing

```javascript
// Create Google search URL
const searchQuery = p.manufacturer_sku || p.product_name;
const googleSearchUrl = `https://www.google.com/search?q=${encodeURIComponent(searchQuery)}`;
```

---

### Node 14: Send HITL Message

**Channel:** `#product-detector`
**Message Type:** Block Kit

Sends individual review requests for each product needing approval.

---

## Slack Integration

### Notification Channels

**Channel:** `#product-detector` (ID: `C0ADK9VMDHC`)

### Summary Notification

Posted when products are detected, includes:
- Total products queued
- Batch size warning (if 100 limit reached)
- Breakdown by status
- Top brands and categories
- Execution ID and timestamp

### HITL Review Messages

Individual messages for each `pending_review` product with:
- Full product details and image
- Interactive buttons for approval workflow
- Google Search button for product research

### Button Actions

All button interactions are handled by [[Slack Integration|TL_Slack_Interaction_Handler]]:
- `approve_product` → Sets status to 'approved'
- `reject_product` → Prompts for rejection reason
- `google_search` → Opens external link (no handler needed)

---

## Mobile Review Workflow

The Google Search button enables efficient mobile product review:

1. Receive HITL notification on mobile
2. Tap **🔍 Google Search** to research product
3. Review specs, images, and competitor pricing
4. Return to Slack and tap **Approve** or **Reject**

No need to copy/paste SKUs or switch between apps.

---

## Error Handling

Workflow errors are logged to `tl_workflow_executions` with status 'failed'. Monitor the execution log in n8n or query Supabase:

```sql
SELECT * FROM tl_workflow_executions
WHERE workflow_name = 'TL_Product_Detector' AND status = 'failed'
ORDER BY started_at DESC;
```

---

## Testing

### Manual Trigger Test

1. Open workflow in n8n
2. Click "Execute Workflow"
3. Check Supabase for new entries in `tl_onboarding_queue`

### Verification Queries

```sql
-- Check new entries were added
SELECT stock_code, product_name, status, priority, detected_at
FROM tl_onboarding_queue
ORDER BY detected_at DESC
LIMIT 20;

-- Check queue summary
SELECT * FROM vw_onboarding_queue_summary;

-- Check workflow logged successfully
SELECT * FROM tl_workflow_executions
WHERE workflow_name = 'TL_Product_Detector'
ORDER BY started_at DESC
LIMIT 5;
```

---

## Maintenance

### Adjusting Detection Limit

Modify the LIMIT in Node 2 to control how many products are queued per run:
- Start with 100 during initial rollout
- Increase to 500+ once process is validated

### Re-Running Detection

The workflow is idempotent due to `ON CONFLICT DO NOTHING`. Safe to re-run at any time.

### Clearing the Queue

```sql
-- Remove all pending_review items (start fresh)
DELETE FROM tl_onboarding_queue
WHERE status = 'pending_review';

-- Reset failed items for retry
UPDATE tl_onboarding_queue
SET status = 'pending_review', retry_count = 0, publish_error = NULL
WHERE status = 'failed';
```

---

## Related Documentation

- [[Workflow 6 - Product Publisher]]
- [[Category Mapping Guide]]
- [[Database Schema - Product Automation]]
- [[System Overview - TechLoop Inventory Automation]]
