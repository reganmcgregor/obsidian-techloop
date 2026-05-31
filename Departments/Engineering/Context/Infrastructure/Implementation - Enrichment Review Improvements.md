# Enrichment Review Improvements - Implementation Guide

**Date:** 2026-02-21
**Status:** Completed
**Last Updated:** 2026-02-22
**Schema Migration:** ✅ Completed

## Overview

This document provides step-by-step instructions for implementing the three enrichment review improvements:
1. Prevent duplicate Slack messages (requires workflow updates)
2. Split multi-value attributes (requires workflow updates)
3. Close match handling (documentation only)

---

## Step 1: Prevent Duplicate Slack Messages

### ✅ 1.1 Schema Migration (COMPLETED)

The following columns have been added to `tl_product_attributes`:
- `sent_to_slack_at` (TIMESTAMPTZ)
- `slack_notification_count` (INTEGER DEFAULT 0)
- Index: `idx_product_attributes_pending_slack`

**Unique constraint updated (2026-02-22):**
- Old: `UNIQUE(wc_product_id, wc_attribute_id)` — only one row per attribute per product
- New: `UNIQUE(wc_product_id, wc_attribute_id, normalized_value)` — multiple values per attribute per product (required for multi-value extraction)
- The old constraint was dropped and replaced. The `ON CONFLICT` clause in the `TL_Enrich_Attributes` upsert (node-9) was updated to match.

### ✅ 1.2 Update TL_Enrichment_Reviewer Workflow

**Workflow ID:** `OarmHP6DpJvAV0Pb`

#### Change 1: Update "Fetch Pending Mappings" Query (node-2) — CTE Approach

**Implementation Note:** Rather than adding a separate "Mark as Sent" node and rewiring workflow connections (as originally planned), the duplicate prevention was implemented using a **CTE (Common Table Expression)** in the existing "Fetch Pending Mappings" node (node-2). This atomically fetches and marks items as sent in a single query, eliminating any race condition window and avoiding workflow rewiring.

The node-2 query was replaced with:

```sql
WITH fetched AS (
  SELECT pa.id
  FROM tl_product_attributes pa
  WHERE pa.review_status = 'pending'
    AND (pa.sent_to_slack_at IS NULL OR pa.sent_to_slack_at < NOW() - INTERVAL '1 day')
  ORDER BY pa.confidence DESC, pa.created_at ASC
  LIMIT 15
),
updated AS (
  UPDATE tl_product_attributes
  SET sent_to_slack_at = NOW(),
      slack_notification_count = slack_notification_count + 1
  WHERE id IN (SELECT id FROM fetched)
  RETURNING id
)
SELECT pa.id, pa.wc_product_id, pa.stock_code, pa.wc_attribute_id,
       pa.raw_value, pa.normalized_value, pa.confidence, pa.source,
       am.name AS attribute_name,
       COALESCE(pm.name, fl.product_name) AS product_name,
       fl.manufacturer_sku
FROM tl_product_attributes pa
JOIN updated u ON u.id = pa.id
JOIN tl_wc_attributes_mirror am ON am.wc_attribute_id = pa.wc_attribute_id
LEFT JOIN tl_wc_products_mirror pm ON pm.wc_id = pa.wc_product_id
LEFT JOIN tl_feed_leader_raw fl ON fl.stock_code = pa.stock_code;
```

**How it works:**
- The `fetched` CTE selects up to 15 pending items that haven't been sent recently
- The `updated` CTE atomically marks those items as sent (`sent_to_slack_at = NOW()`)
- The final SELECT returns the full row data joined to the updated IDs
- When there are no pending items, the Postgres node returns `{"success": true}` (not an empty array)

#### Change 2: Update IF Has Items Node (node-3)

Because the CTE returns `{"success": true}` when there are no pending items (rather than an empty array), the IF Has Items condition was updated to check for a valid `id` field rather than just item count:

**Condition:** `i.json.id !== undefined && i.json.id !== null`

This correctly routes to the "no items" branch when the CTE result is the `{"success": true}` object.

#### Change 3: Update Format Summary Node (node-9)

The Format Summary node was updated to guard against the case where "Build Review Messages" did not execute (i.e., no items were found):

**Expression updated to:** `$if($('Build Review Messages').isExecuted, <summary_expression>, 'No items to review')`

### 📋 1.3 Backfill Existing Pending Items

**Action:** Run this SQL in Supabase SQL Editor (or via SSH as supabase_admin):

```sql
-- Mark all current pending items as "already sent" to avoid re-sending on first run
UPDATE tl_product_attributes
SET sent_to_slack_at = NOW() - INTERVAL '1 hour',
    slack_notification_count = 1
WHERE review_status = 'pending';
```

**Purpose:** Prevents existing pending items from being re-sent when the updated workflow first runs.

---

## Step 2: Split Multi-Value Attributes

### 🔧 2.1 Update TL_Enrich_Attributes Workflow

**Workflow ID:** `lq8960K0xVwF3Xst`

#### Change 1: Update "Parse LLM Response" (node-8)

**Current Location:** Code node, likely named "Parse LLM Response"

**Action:** Replace the entire code with the enhanced version below:

```javascript
const items = $input.all();
const attrDict = $('Fetch WC Attributes Dictionary').all()[0].json;

// Build attribute map and alias lookup
const wcAttrs = {};
const aliasLookup = {};
for (const attr of attrDict) {
  wcAttrs[attr.attribute_name.toLowerCase()] = attr;
  if (Array.isArray(attr.aliases)) {
    for (const alias of attr.aliases) {
      aliasLookup[alias.toLowerCase()] = attr.attribute_name;
    }
  }
}

const allExtractions = [];

for (const item of items) {
  const llmOutput = item.json.output;
  let parsed;
  try {
    parsed = JSON.parse(llmOutput);
  } catch (e) {
    continue;
  }

  if (!Array.isArray(parsed)) parsed = [parsed];

  for (const extraction of parsed) {
    let attrName = extraction.attribute_name || '';
    let rawValue = extraction.raw_value || '';
    let normalized = extraction.normalized_value || rawValue;

    // Resolve alias to canonical attribute name
    const resolvedAttr = aliasLookup[attrName.toLowerCase()] || attrName;
    const wcAttr = wcAttrs[resolvedAttr.toLowerCase()];
    if (!wcAttr) continue;

    // ========== NEW: MULTI-VALUE PATTERN DETECTION ==========
    const values = splitMultiValue(normalized);

    for (const val of values) {
      // Check for duplicates before adding
      const isDupe = allExtractions.some(e =>
        e.wc_attribute_id === wcAttr.wc_attribute_id &&
        e.normalized_value.toLowerCase() === val.toLowerCase()
      );
      if (isDupe) continue;

      allExtractions.push({
        wc_attribute_id: wcAttr.wc_attribute_id,
        raw_value: rawValue,
        normalized_value: val,
        confidence: extraction.confidence || 0.90
      });
    }
  }
}

return [{ json: { extractions: allExtractions } }];

// ========== NEW HELPER FUNCTION ==========
function splitMultiValue(value) {
  // Pattern 1: "Yes x N (...)" repeated (e.g., "Yes x 1 (DVI-D) Yes x 1 (HDMI 2.1)")
  const yesXPattern = /Yes\s*x\s*\d+\s*\(([^)]+)\)/gi;
  const matches = [];
  let match;
  while ((match = yesXPattern.exec(value)) !== null) {
    matches.push(match[1].trim());
  }
  if (matches.length > 1) return matches;

  // Pattern 2: Multiple parenthetical groups (e.g., "(DVI-D) (HDMI) (DisplayPort)")
  const parenPattern = /\(([^)]+)\)/g;
  const parenMatches = [];
  while ((match = parenPattern.exec(value)) !== null) {
    parenMatches.push(match[1].trim());
  }
  if (parenMatches.length > 1) return parenMatches;

  // Pattern 3: Comma-separated fallback (existing logic)
  if (value.includes(',')) {
    const parts = value.split(',').map(p => p.trim());
    // Only split if each part is < 30 chars (avoid splitting long descriptive text)
    if (parts.every(p => p.length < 30 && p.length > 0)) {
      return parts;
    }
  }

  // Single value - return as-is
  return [value];
}
```

**Key Changes:**
- Added `splitMultiValue()` helper function at the bottom
- Replaced single-value extraction with `const values = splitMultiValue(normalized);` loop
- Detects patterns: "Yes x N (...)", multiple "(…)" groups, comma-separated values

#### Change 2: Update "Build LLM Prompts" (node-6)

**Current Location:** Code node, likely named "Build LLM Prompts"

**Action:** Add the following instruction to the CRITICAL section of the LLM prompt template:

```javascript
// Find the section where the prompt is constructed
// Add this text to the CRITICAL instructions section:

CRITICAL: Each JSON object must have exactly ONE value. If a product has multiple values for an attribute:

CORRECT:
[
  {"attribute_name": "Interface", "normalized_value": "Native DVI-D", ...},
  {"attribute_name": "Interface", "normalized_value": "Native HDMI 2.1", ...}
]

WRONG:
[
  {"attribute_name": "Interface", "normalized_value": "Yes x 1 (Native DVI-D) Yes x 1 (Native HDMI 2.1)", ...}
]

If you see patterns like "Yes x 1 (...) Yes x 2 (...)", extract each parenthetical group as a separate object.
```

**Note:** Locate the section in the code where the prompt string is built (likely a template literal or string concatenation) and insert this instruction in the CRITICAL section.

---

## Step 3: Close Match Handling

**Status:** Documentation only - no code changes needed

The existing alias system ("Map to Existing" dropdown) already solves this problem. The solution is user training + optional pre-seeding of common aliases.

### 📖 Best Practices Documentation

See [[Enrichment Review - Best Practices]] — created and complete.

### 🌱 Optional: Pre-Seed Common Aliases

**Run via Supabase SQL Editor:**

```sql
-- First, get attribute IDs
SELECT wc_attribute_id, name FROM tl_wc_attributes_mirror
WHERE name IN ('Screen Size', 'RAM', 'Storage Capacity', 'Motherboard Form Factor');

-- Then insert aliases (replace <attr_id> with actual IDs from above query)
INSERT INTO tl_attribute_aliases (alias_name, maps_to_wc_attribute_id, source) VALUES
('inches', '<screen_size_attr_id>', 'manual'),
('inch', '<screen_size_attr_id>', 'manual'),
('GB', '<ram_attr_id>', 'manual'),
('Gigabytes', '<ram_attr_id>', 'manual'),
('Terabyte', '<storage_attr_id>', 'manual'),
('TB', '<storage_attr_id>', 'manual'),
('M/B', '<motherboard_attr_id>', 'manual')
ON CONFLICT (alias_name) DO NOTHING;
```

---

## Verification Plan

### Test 1: Duplicate Messages Fixed

1. Insert 3 test rows:
```sql
INSERT INTO tl_product_attributes (wc_product_id, stock_code, wc_attribute_id, raw_value, normalized_value, review_status, confidence)
VALUES
  (24291, 'TEST-001', (SELECT wc_attribute_id FROM tl_wc_attributes_mirror WHERE name = 'RAM' LIMIT 1), '16GB', '16GB', 'pending', 0.95),
  (24291, 'TEST-002', (SELECT wc_attribute_id FROM tl_wc_attributes_mirror WHERE name = 'RAM' LIMIT 1), '32GB', '32GB', 'pending', 0.92),
  (24291, 'TEST-003', (SELECT wc_attribute_id FROM tl_wc_attributes_mirror WHERE name = 'Storage Capacity' LIMIT 1), '1TB', '1TB', 'pending', 0.88);
```

2. Run `TL_Enrichment_Reviewer` manually
3. Verify 3 messages appear in `#enrichment-review`
4. Check DB: all should have `sent_to_slack_at` set, `slack_notification_count = 1`
5. Run workflow again 5 minutes later → NO new messages
6. Set one row's `sent_to_slack_at = NOW() - INTERVAL '2 days'` → run workflow → ONE reminder

### Test 2: Multi-Value Splitting

1. Find product with "Interface: Yes x 1 (DVI-D) Yes x 1 (HDMI 2.1)" in specs
2. Reset that product's `attribute_status = 'pending'`
3. Run `TL_Enrich_Attributes`
4. Query: `SELECT * FROM tl_product_attributes WHERE stock_code = '<sku>' AND wc_attribute_id = (SELECT wc_attribute_id FROM tl_wc_attributes_mirror WHERE name = 'Interface')`
5. Verify TWO rows: one "DVI-D", one "HDMI 2.1"

### Test 3: Alias System

1. In Slack `#enrichment-review`, use "Map to Existing" dropdown for close match
2. Verify alias created: `SELECT * FROM tl_attribute_aliases WHERE alias_name = '<the_proposed_term>'`
3. Re-run extraction for another product with same term → verify auto-resolved (no new proposal)

---

## Completion Status

All implementation items completed as of 2026-02-22:

**Schema (completed 2026-02-21)**
- Added `sent_to_slack_at` and `slack_notification_count` to `tl_product_attributes`
- Created index `idx_product_attributes_pending_slack`
- Updated unique constraint from `UNIQUE(wc_product_id, wc_attribute_id)` to `UNIQUE(wc_product_id, wc_attribute_id, normalized_value)`

**TL_Enrichment_Reviewer workflow (completed 2026-02-22)**
- Updated Fetch Pending Mappings (node-2) with atomic CTE approach
- Updated IF Has Items (node-3) to check for `id` field
- Updated Format Summary (node-9) with `isExecuted` guard
- Added `onError: "continueRegularOutput"` to Slack Summary (node-10)

**TL_Enrich_Attributes workflow (completed 2026-02-22)**
- Updated Parse LLM Response (node-8) with `splitMultiValue()` helper
- Updated Build LLM Prompts (node-6) with explicit multi-value instruction
- Updated Upsert Extractions (node-9) `ON CONFLICT` to match new unique constraint

**Documentation (completed 2026-02-22)**
- Created [[Enrichment Review - Best Practices]]
- Updated [[Database Schema - Product Automation]] (unique constraint, last updated)
- Updated [[Phase 6.3 - Product Enrichment Pipeline]] (improvements noted)
