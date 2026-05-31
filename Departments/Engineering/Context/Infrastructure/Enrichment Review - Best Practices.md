# Enrichment Review Best Practices

**Purpose:** Guidelines for reviewing LLM-proposed product attributes in the `#enrichment-review` Slack channel.

**Related:** [[Implementation - Enrichment Review Improvements|Implementation Guide]] | [[Phase 6.3 - Product Enrichment Pipeline]]

---

## Overview

The Product Enrichment Pipeline uses LLMs to extract product attributes from manufacturer specifications. Human review ensures quality before attributes are pushed to WooCommerce.

**Review Channels:**
- `#enrichment-review` - Attribute mappings (existing attributes, proposed terms)
- `#product-publisher` - Complete products ready for publishing
- `#queue-review` - Raw product queue processing

This guide focuses on **`#enrichment-review`** workflows.

---

## Handling Close Matches

### The Problem

The LLM frequently suggests terms that are **almost identical** to existing WooCommerce terms:

**Examples:**
- LLM proposes: `15.6 inches` | WC has: `15.6"`
- LLM proposes: `16 Gigabytes` | WC has: `16GB`
- LLM proposes: `Micro ATX` | WC has: `MicroATX`
- LLM proposes: `M/B` | WC has: `Motherboard`

### ✅ Correct Approach: Pre-Seed Aliases via SQL

The **TL_Enrichment_Reviewer** only has Approve and Reject buttons — there is no "Map to Existing" dropdown here. For close matches, the best approach is to **pre-seed aliases** before they reach the review queue.

See the **Pre-Seeding Common Aliases** section below for SQL instructions.

If a close match has already appeared in your queue:
- **Reject it** (it's the wrong format)
- **Then add the alias** via SQL so future extractions auto-resolve correctly

### ❌ What NOT to Do

**DON'T: Approve the close match as a new term**
- Creates duplicate terms in WooCommerce (e.g., both `15.6"` and `15.6 inches`)
- Fragments product filtering (some products tagged with one, some with the other)
- Harder to maintain taxonomy consistency

**DON'T: Hope rejecting it will fix the pattern**
- Rejected items are **dead-end** - they don't re-queue
- The same close match will appear again on the next extraction run
- You must seed the alias to prevent recurrence

### ℹ️ "Map to Existing" — Where It Actually Lives

The **"Map to Existing"** dropdown exists in **TL_Attribute_Proposer** (the `#enrichment-review` channel, separate workflow). It's used when the LLM proposes an entirely **new attribute name** that should instead be mapped to an existing WooCommerce attribute (e.g., LLM proposes "M/B" as a new attribute — you map it to "Motherboard").

This is different from the term-value close matches handled by TL_Enrichment_Reviewer.

---

## Common Unit Normalisation Patterns

These are the most frequent close matches you'll encounter:

### Screen Size
- **Canonical:** `15.6"`, `17.3"`, `24"`
- **Common aliases:** `15.6 inches`, `15.6 inch`, `15.6-inch`
- **Action:** Seed aliases → canonical quoted format (via SQL)

### Storage Capacity
- **Canonical:** `1TB`, `512GB`, `2TB`
- **Common aliases:** `1 Terabyte`, `512 Gigabytes`, `1TB SSD`
- **Action:** Seed aliases → canonical abbreviation without spaces (via SQL)

### RAM / Memory
- **Canonical:** `16GB`, `32GB`, `64GB`
- **Common aliases:** `16 GB`, `16GB DDR4`, `16 Gigabytes`
- **Action:** Seed aliases → canonical abbreviation without spaces (via SQL)

### Form Factors
- **Canonical:** `MicroATX`, `Mini-ITX`, `ATX`
- **Common aliases:** `Micro ATX`, `microATX`, `M-ATX`
- **Action:** Seed aliases → canonical capitalisation (via SQL)

### Abbreviations
- **Canonical:** `Motherboard`, `Wi-Fi`, `Bluetooth`
- **Common aliases:** `M/B`, `WiFi`, `BT`
- **Action:** Seed aliases → full canonical term (via SQL)

---

## Review Actions Explained

**TL_Enrichment_Reviewer** Slack notifications have two action buttons:

### 1. Approve

**When to use:**
- The term is **exactly correct** and doesn't already exist in WooCommerce
- The term is **properly normalised** (consistent format)
- You're **confident** this should be a new term

**What happens:**
- `review_status` → `approved`
- Next workflow run: pushed to WooCommerce as new term
- Future extractions: this term becomes part of the LLM dictionary

### 2. Reject

**When to use:**
- The term is **factually incorrect** (LLM hallucinated or misinterpreted)
- The term is **too generic** ("Yes", "No", "Standard")
- The attribute **doesn't apply** to this product category
- The term is a close match to an existing term (wrong format) — then **also seed an alias** via SQL

**What happens:**
- `review_status` → `rejected`
- Item is **permanently removed** from queue
- Does NOT re-queue for future review

**⚠️ After rejecting a close match:** Add an alias via SQL (see Pre-Seeding section) to prevent the same variant reappearing.

---

## Workflow FAQs

### Q: Will rejected items ever come back?

**A:** No. Once rejected, `review_status = 'rejected'` items are excluded from future queries. They're a dead-end.

### Q: What if I accidentally reject something?

**A:** You'll need to manually update the database:
```sql
UPDATE tl_product_attributes
SET review_status = 'pending'
WHERE id = '<the-rejected-item-id>';
```

Or ask the LLM to re-extract that product's attributes.

### Q: How does the system improve over time?

**A:** Two mechanisms:
1. **Approved terms** → added to LLM prompt dictionary
2. **Manually seeded aliases** → included in prompt as "also known as"

Approved terms expand the LLM's vocabulary. Pre-seeded aliases prevent close-match variants from ever reaching the review queue.

### Q: Can I see all aliases for an attribute?

**A:** Yes, via SQL:
```sql
SELECT aa.alias_name, am.name AS canonical_attribute
FROM tl_attribute_aliases aa
JOIN tl_wc_attributes_mirror am ON am.wc_attribute_id = aa.maps_to_wc_attribute_id
WHERE am.name = 'Screen Size'
ORDER BY aa.alias_name;
```

Or check the `aliases` field in the `Fetch WC Attributes Dictionary` node output.

### Q: Why do I see the same notification multiple times?

**A:** As of 2026-02-21, the workflow sends reminders for items stuck in `pending` status for 24+ hours. If you see duplicates within 24 hours, this is a bug - check the `sent_to_slack_at` timestamp in the database.

### Q: What if the LLM suggests multiple values as one term?

**Example:** `"Interface: Yes x 1 (DVI-D) Yes x 1 (HDMI 2.1)"` as a single term instead of two separate terms.

**A:** This is a known pattern (especially in manufacturer specs). The parser now includes multi-value splitting logic:
- Detects "Yes x N (...)" patterns
- Detects multiple parenthetical groups
- Falls back to comma-splitting for other cases

If you still see compound values, reject the notification and let the LLM re-extract (the prompt has been updated to prevent this).

---

## Advanced: Pre-Seeding Common Aliases

If you notice the same close matches appearing frequently, you can **pre-seed aliases** to prevent them from ever reaching review.

**Example:** Pre-seed common unit aliases:

```sql
-- First, get attribute IDs
SELECT wc_attribute_id, name FROM tl_wc_attributes_mirror
WHERE name IN ('Screen Size', 'RAM', 'Storage Capacity');

-- Then insert aliases (replace <attr_id> with actual IDs)
INSERT INTO tl_attribute_aliases (alias_name, maps_to_wc_attribute_id, source) VALUES
('inches', '<screen_size_attr_id>', 'manual'),
('inch', '<screen_size_attr_id>', 'manual'),
('GB', '<ram_attr_id>', 'manual'),
('Gigabytes', '<ram_attr_id>', 'manual'),
('Terabyte', '<storage_attr_id>', 'manual'),
('TB', '<storage_attr_id>', 'manual')
ON CONFLICT (alias_name) DO NOTHING;
```

**Result:** Any future extraction of "15.6 inches" automatically maps to the canonical term **before** it reaches your Slack review queue.

---

## Quick Reference Card

| Scenario | Action | Result |
|----------|--------|--------|
| **New term, correctly formatted** | Approve | New WC term created |
| **Close match (unit/format variation)** | Reject + seed alias via SQL | Term rejected; alias prevents recurrence |
| **Factually incorrect** | Reject | Permanently removed |
| **Too generic / not applicable** | Reject | Permanently removed |
| **Multiple values in one term** | Reject | Let LLM re-extract (parser will split) |

---

## Related Documentation

- [[Implementation - Enrichment Review Improvements]] - Technical implementation details
- [[Phase 6.3 - Product Enrichment Pipeline]] - Pipeline architecture
- [[Database Schema - Product Automation]] - Schema reference
- [[TL_Enrichment_Reviewer - Workflow Docs]] - Slack notification workflow
- [[TL_Slack_Interaction_Handler - Workflow Docs]] - Button handler workflow

---

**Last Updated:** 2026-02-21
**Maintainer:** Regan McGregor
