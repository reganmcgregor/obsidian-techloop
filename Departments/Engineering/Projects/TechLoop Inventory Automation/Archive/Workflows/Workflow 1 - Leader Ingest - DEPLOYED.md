> [!info] Part of [[Master Note - Inventory Automation]]

# Workflow 1: Leader Ingest - DEPLOYED

**Status:** ✅ Deployed (Inactive)
**Workflow ID:** `mUkhS0BfN7Lq6EsS`
**Schedule:** Every 2 hours at :00 (cron: `0 */2 * * *`)
**Created:** 2026-01-13
**Last Updated:** 2026-01-13

---

## Overview

The **TL_Ingest_Leader_Feed** workflow fetches the Leader XML product feed every 2 hours, transforms the data to snake_case, sanitizes availability fields, and bulk upserts all products to the `tl_feed_leader_raw` Supabase table.

### Purpose

- Ingest ALL raw data from Leader supplier feed (50+ fields per product)
- Maintain fresh product availability and pricing data
- Track discontinuation via `last_seen_at` timestamp
- Provide execution logging and email reporting

### Key Features

- **Scheduled Execution:** Every 2 hours at :00 (:00, 2:00, 4:00, 6:00, etc.)
- **Bulk Processing:** Handles 10,000+ products efficiently
- **Data Sanitization:** Converts "CALL"/"POA" to 0 for availability_total
- **Snake Case Transform:** All XML tags converted to snake_case for PostgreSQL
- **Execution Logging:** Every run logged to `tl_workflow_executions`
- **Email Reporting:** Summary email sent to regan@techloop.com.au

---

## Workflow Structure

### Node Flow

```
1. Schedule Every 2 Hours (Schedule Trigger)
   ↓
2. Fetch Leader XML Feed (HTTP Request)
   ↓
3. Parse XML to JSON (XML Parser)
   ↓
4. Transform and Batch (Code Node)
   ↓
5. Upsert to Supabase (Code Node)
   ↓
6. Supabase Insert (Supabase Node)
   ↓
7. Prepare Summary (Code Node)
   ↓
8. Log to tl_workflow_executions (Supabase Node)
   ↓
9. Prepare Email (Code Node)
   ↓
10. Send Email (Email Send Node)
```

### Node Details

#### 1. Schedule Every 2 Hours
- **Type:** Schedule Trigger
- **Cron Expression:** `0 */2 * * *`
- **Timezone:** Australia/Sydney
- **Purpose:** Triggers workflow every 2 hours on the hour

#### 2. Fetch Leader XML Feed
- **Type:** HTTP Request
- **Method:** GET
- **URL:** `https://partner.leadersystems.com.au/WSDataFeed.asmx/DownLoad?CustomerCode=U2FsdGVkX19iL70lfWvWzLkZdA25pj1GfrS5R8itZoTxERYgWe23r%2BKRKDpCvblP&WithHeading=true&WithLongDescription=true&DataType=1`
- **Timeout:** 120 seconds
- **Purpose:** Downloads Leader XML feed (typically 10MB+, 10,000+ products)

#### 3. Parse XML to JSON
- **Type:** XML Parser
- **Mode:** XML to JSON
- **Options:**
  - `explicitArray: false` - Single items not wrapped in array
  - `trim: true` - Remove whitespace
  - `normalize: true` - Normalize text content
- **Purpose:** Converts XML feed to JSON structure

#### 4. Transform and Batch
- **Type:** Code Node (JavaScript)
- **Purpose:**
  - Transforms all XML tags to snake_case
  - Sanitizes `availability_total` ("CALL"/"POA" → 0)
  - Sanitizes numeric fields (dbp, rrp, weight, etc.)
  - Adds `last_seen_at` timestamp
  - Creates batches of 50 items (not currently used but prepared for batching)

**Key Transformations:**
- `StockCode` → `stock_code`
- `BarCode` → `bar_code`
- `ProductName` → `product_name`
- `AvailabilityTotal` → `availability_total` (sanitized)
- Raw availability stored as `availability_total_raw`

#### 5. Upsert to Supabase
- **Type:** Code Node (JavaScript)
- **Purpose:**
  - Flattens all batches into individual items
  - Maps all 50+ fields to database columns
  - Prepares data structure for Supabase upsert
  - Stores full raw data in `raw_data` JSONB field

**Mapped Fields:**
- Core: stock_code, bar_code, manufacturer_sku
- Product: product_name, product_name2, manufacturer, category_code, category_name, subcategory_name
- Pricing: dbp, dbp5, rrp, standard_rrp
- Availability: availability_total, availability_total_raw, availability_nsw/qld/vic/wa/sa
- ETAs: eta_nsw/qld/vic/wa/sa
- Physical: weight, length, width, height, warranty_length
- Media: image_url, image_filename, website_url
- Tracking: last_seen_at, raw_data

#### 6. Supabase Insert
- **Type:** Supabase Node
- **Operation:** Create (with upsert)
- **Table:** `tl_feed_leader_raw`
- **Conflict Column:** `stock_code`
- **Data Mode:** Auto Map Input Data
- **Purpose:** Bulk upsert all products to database

**Upsert Behavior:**
- If `stock_code` exists → UPDATE all fields
- If `stock_code` is new → INSERT new record
- Updates `last_seen_at` timestamp for discontinuation tracking

#### 7. Prepare Summary
- **Type:** Code Node (JavaScript)
- **Purpose:**
  - Aggregates Supabase results
  - Counts successes and failures
  - Calculates execution duration
  - Prepares data for logging and email

**Output Structure:**
```json
{
  "workflow_name": "TL_Ingest_Leader_Feed",
  "execution_id": "...",
  "started_at": "2026-01-13T04:00:00.000Z",
  "completed_at": "2026-01-13T04:02:30.000Z",
  "duration_seconds": 150,
  "status": "success",
  "total_items": 10432,
  "success_count": 10432,
  "fail_count": 0,
  "error_messages": null,
  "metadata": {
    "batch_size": 50,
    "feed_source": "Leader XML Feed"
  }
}
```

#### 8. Log to tl_workflow_executions
- **Type:** Supabase Node
- **Operation:** Create
- **Table:** `tl_workflow_executions`
- **Purpose:** Persist execution log for monitoring and troubleshooting

#### 9. Prepare Email
- **Type:** Code Node (JavaScript)
- **Purpose:** Generates HTML and plain text email report

**Email Features:**
- Status-based color coding (green/yellow/red)
- Summary statistics (total, success, failed, duration)
- Australian datetime formatting (Australia/Sydney timezone)
- Mobile-responsive HTML design
- Error details (if any)

#### 10. Send Email
- **Type:** Email Send Node
- **From:** noreply@techloop.com.au
- **To:** regan@techloop.com.au
- **Format:** Both HTML and Plain Text
- **SMTP Credentials:** `vkCbG62NopBnWPHl` (Elastic Email)
- **Purpose:** Send execution summary report

**Email Subject Format:**
- Success: `✅ Leader Feed: 10,432 products ingested (success)`
- Partial: `⚠️ Leader Feed: 10,200 products ingested (partial_success)`
- Failed: `❌ Leader Feed: 0 products ingested (failed)`

---

## Data Flow

### Input
- **Source:** Leader XML feed (RSS format)
- **Size:** ~10-15MB XML
- **Products:** ~10,000-15,000 items
- **Update Frequency:** Real-time from Leader's systems

### Processing
1. **XML Parsing:** 10-20 seconds
2. **Transformation:** 30-60 seconds (all products)
3. **Supabase Upsert:** 60-120 seconds (bulk insert)
4. **Logging & Email:** 5-10 seconds

**Total Duration:** 2-4 minutes for full feed

### Output
- **Supabase Table:** `tl_feed_leader_raw` (upserted)
- **Execution Log:** `tl_workflow_executions` (inserted)
- **Email Report:** Sent to regan@techloop.com.au

---

## Configuration

### Schedule
- **Frequency:** Every 2 hours
- **Cron:** `0 */2 * * *`
- **Timezone:** Australia/Sydney
- **Times:** 12:00 AM, 2:00 AM, 4:00 AM, 6:00 AM, 8:00 AM, 10:00 AM, 12:00 PM, 2:00 PM, 4:00 PM, 6:00 PM, 8:00 PM, 10:00 PM
- **Daily Runs:** 12 executions per day

### Timeouts
- **HTTP Request:** 120 seconds (2 minutes)
- **Workflow Execution:** 600 seconds (10 minutes)

### Error Handling
- **Execution Errors:** Saved to n8n execution log
- **Success Data:** Saved to n8n execution log
- **Manual Executions:** Saved to n8n execution log
- **Failed Items:** Logged to `tl_workflow_executions` with error details

### Credentials Used
1. **Supabase:** `Supabase account` (ID: 1)
   - API URL: `https://supabase.reganmcgregor.com.au`
   - Used for database operations
2. **SMTP:** `SMTP account` (ID: vkCbG62NopBnWPHl)
   - Service: Elastic Email
   - Used for sending email reports

---

## Activation Instructions

### Prerequisites
1. ✅ Supabase table `tl_feed_leader_raw` exists with correct schema
2. ✅ Supabase table `tl_workflow_executions` exists with correct schema
3. ✅ Supabase credentials configured in n8n (ID: 1)
4. ✅ SMTP credentials configured in n8n (ID: vkCbG62NopBnWPHl)
5. ⚠️ Test execution completed successfully (RECOMMENDED)

### Activation Steps

**Option 1: Via n8n Web UI**
1. Open n8n: https://n8n.reganmcgregor.com.au
2. Navigate to workflow: `TL_Ingest_Leader_Feed` (ID: mUkhS0BfN7Lq6EsS)
3. Click **Activate** toggle in top right
4. Verify schedule is set to `0 */2 * * *`
5. Wait for next :00 hour or trigger manually for testing

**Option 2: Via n8n MCP Tool**
```javascript
// Use n8n_update_partial_workflow
{
  "id": "mUkhS0BfN7Lq6EsS",
  "operations": [{
    "type": "activateWorkflow"
  }]
}
```

### Testing Before Activation

**Manual Test Execution:**
1. Open workflow in n8n Web UI
2. Click **Test workflow** button
3. Monitor execution in real-time
4. Verify:
   - XML feed fetched successfully
   - Products transformed correctly
   - Supabase upsert completes
   - Execution logged to database
   - Email received

**Expected Results:**
- Duration: 2-4 minutes
- Products ingested: 10,000-15,000
- Email received with success status
- Database record in `tl_workflow_executions`

---

## Monitoring

### Key Metrics

**Success Indicators:**
- ✅ Status = "success"
- ✅ Duration < 5 minutes
- ✅ Total items > 9,000
- ✅ Fail count = 0
- ✅ Email received within 5 minutes of hour

**Warning Indicators:**
- ⚠️ Status = "partial_success"
- ⚠️ Duration > 5 minutes
- ⚠️ Fail count > 0 but < 10%
- ⚠️ Total items < 9,000

**Error Indicators:**
- ❌ Status = "failed"
- ❌ Duration > 10 minutes (timeout)
- ❌ Fail count > 10%
- ❌ No email received
- ❌ HTTP request failed (Leader feed down)

### Monitoring Queries

**Check Recent Executions:**
```sql
SELECT workflow_name, started_at, status, duration_seconds, total_items, fail_count
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Ingest_Leader_Feed'
ORDER BY started_at DESC
LIMIT 10;
```

**Check Daily Success Rate:**
```sql
SELECT
  DATE(started_at) as date,
  COUNT(*) as total_runs,
  COUNT(*) FILTER (WHERE status = 'success') as successes,
  ROUND(AVG(duration_seconds), 0) as avg_duration,
  SUM(total_items) as total_products_ingested
FROM tl_workflow_executions
WHERE workflow_name = 'TL_Ingest_Leader_Feed'
  AND started_at > NOW() - INTERVAL '7 days'
GROUP BY DATE(started_at)
ORDER BY date DESC;
```

**Check for Discontinued Products:**
```sql
SELECT stock_code, product_name, manufacturer, last_seen_at,
  NOW() - last_seen_at as time_since_last_seen
FROM tl_feed_leader_raw
WHERE last_seen_at < NOW() - INTERVAL '4 hours'
ORDER BY last_seen_at ASC
LIMIT 20;
```

### Email Reports

**Report Content:**
- Execution status (success/partial/failed)
- Total products processed
- Success/failure counts
- Execution duration
- Start and completion timestamps (Sydney time)
- Error messages (if any)
- Execution ID for troubleshooting

**Delivery:**
- Sent after every execution
- To: regan@techloop.com.au
- From: noreply@techloop.com.au
- Format: HTML + Plain Text

---

## Troubleshooting

### Issue: Workflow Not Running

**Symptoms:**
- No email reports received
- No new records in `tl_workflow_executions`
- Last execution more than 2 hours ago

**Diagnosis:**
1. Check workflow active status in n8n UI
2. Verify schedule trigger: `0 */2 * * *`
3. Check n8n service status: `systemctl status n8n`
4. Review n8n server logs

**Resolution:**
- Reactivate workflow in n8n UI
- Restart n8n service if needed
- Verify server clock is correct

---

### Issue: HTTP Request Timeout

**Symptoms:**
- Error: "Request timeout"
- Duration close to 120 seconds
- No products ingested

**Diagnosis:**
1. Test Leader feed URL manually
2. Check network connectivity from n8n server
3. Review Leader feed size (may have grown significantly)

**Resolution:**
- Increase timeout in "Fetch Leader XML Feed" node (current: 120s)
- Contact Leader support if feed is consistently slow
- Check n8n server internet connection

---

### Issue: XML Parsing Error

**Symptoms:**
- Error in "Parse XML to JSON" node
- Message: "Invalid XML"
- No products processed

**Diagnosis:**
1. Download XML feed manually and inspect
2. Check for malformed XML tags
3. Verify feed URL is correct

**Resolution:**
- Leader may have changed feed format
- Update XML parser options if needed
- Contact Leader support for feed format documentation

---

### Issue: Supabase Upsert Failures

**Symptoms:**
- Status = "partial_success"
- Fail count > 0
- Error messages in email

**Diagnosis:**
1. Check Supabase database connectivity
2. Review error messages in `tl_workflow_executions`
3. Verify table schema matches expected fields
4. Check for data type mismatches

**Resolution:**
- Fix data transformation logic in "Upsert to Supabase" node
- Update database schema if new fields added
- Handle null values properly

---

### Issue: Email Not Received

**Symptoms:**
- Execution completes successfully
- Log shows email sent
- No email in inbox

**Diagnosis:**
1. Check spam/junk folder
2. Verify SMTP credentials (Elastic Email)
3. Review n8n email send logs
4. Check Elastic Email dashboard for bounces

**Resolution:**
- Verify email address: regan@techloop.com.au
- Check SMTP credential validity
- Test SMTP connection independently
- Review Elastic Email account status

---

### Issue: Products Not Updating

**Symptoms:**
- Workflow runs successfully
- Products in database but not reflecting latest feed
- `last_seen_at` not updating

**Diagnosis:**
1. Check if upsert is working (conflict on `stock_code`)
2. Verify `last_seen_at` field is being set in transform
3. Compare database records with raw XML feed

**Resolution:**
- Ensure `stock_code` is unique primary key
- Verify upsert ON CONFLICT clause
- Check data transformation includes `last_seen_at`

---

## Performance Optimization

### Current Performance
- **Feed Size:** ~10-15MB XML
- **Products:** ~10,000-15,000 items
- **Parse Time:** 10-20 seconds
- **Transform Time:** 30-60 seconds
- **Upsert Time:** 60-120 seconds
- **Total Duration:** 2-4 minutes

### Optimization Opportunities

**If Duration Exceeds 5 Minutes:**
1. Implement true batch processing (50 items per batch with loop)
2. Use Supabase's batch insert API endpoint
3. Parallel processing for large batches
4. Increase n8n workflow memory limits

**If Feed Size Grows Significantly:**
1. Implement incremental sync (if Leader provides lastModified)
2. Use streaming XML parser instead of loading full document
3. Process only changed products

**Database Optimization:**
1. Ensure indexes on `stock_code` (primary key)
2. Add index on `last_seen_at` for discontinuation queries
3. Partition table by date if historical data grows large
4. Run VACUUM ANALYZE weekly

---

## Related Documentation

- [[Database Schema - TechLoop Inventory]] - Full database schema
- [[Master Note - Inventory Automation]] - Project overview
- [[Phase 2 - Ingestion]] - Implementation notes
- `robust-forging-blossom.md` - Original implementation plan

---

## Changelog

### 2026-01-13 - Initial Deployment
- Created workflow with 10 nodes
- Configured schedule: Every 2 hours at :00
- Implemented data transformation and sanitization
- Added execution logging and email reporting
- Status: Ready for activation
- **NOT YET ACTIVATED** - Requires testing

---

## Next Steps

1. ✅ **COMPLETE** - Workflow created and configured
2. ⏳ **TODO** - Test manual execution
3. ⏳ **TODO** - Verify Supabase upsert works correctly
4. ⏳ **TODO** - Verify email report received
5. ⏳ **TODO** - Activate workflow
6. ⏳ **TODO** - Monitor first 24 hours (12 executions)
7. ⏳ **TODO** - Build Workflow 2: WooCommerce Mirroring

---

## Notes

- Workflow is currently **INACTIVE** - must be manually activated
- All code nodes use JavaScript (not Python)
- Timezone set to Australia/Sydney for schedule and email times
- Email credentials use Elastic Email SMTP service
- Supabase credentials point to https://supabase.reganmcgregor.com.au
- First execution after activation will process ALL products (full sync)
- Subsequent executions update existing products via upsert
- `last_seen_at` timestamp enables discontinuation detection in Workflow 3
