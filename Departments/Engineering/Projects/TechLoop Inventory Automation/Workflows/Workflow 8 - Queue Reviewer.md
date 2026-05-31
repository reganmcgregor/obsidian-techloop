# Workflow 7: Queue Reviewer (TL_Queue_Reviewer)

**Status:** Deployed
**n8n Name:** `TL_Queue_Reviewer`
**n8n ID:** `I0eOOo8geZfhLKfs`
**Direct Link:** [Open in n8n](https://n8n.reganmcgregor.com.au/workflow/I0eOOo8geZfhLKfs)
**Schedule:** Every 4 hours
**Slash Command:** `/review-queue` (Path: `/webhook/slack/review-queue`)

---

## Purpose

The Review Sampler periodically (or on-demand) pulls a random set of 5 products from the `pending_review` queue and sends them to Slack for human review. This helps process the backlog of thousands of products in manageable chunks.

---

## Workflow Logic

1. **Trigger:** Runs every 4 hours or when the `/review-queue` slash command is used.
2. **Fetch:** Queries Supabase for 5 random products where `status = 'pending_review'`.
3. **Format:** Constructs Slack Block Kit messages with product details and images.
4. **Interact:** Includes "Approve" and "Reject" buttons.
5. **Handle:** Responses are processed by [[Slack Integration|TL_Slack_Interaction_Handler]].

---

## Node Architecture

| # | Node Type | Node Name | Purpose |
|---|-----------|-----------|---------|
| 1 | Schedule Trigger | ScheduleTrigger | Runs every 4 hours |
| 2 | Webhook | SlackSlashCommand | Supports `/review-queue` |
| 3 | Supabase | FetchProducts | `SELECT * FROM tl_onboarding_queue WHERE status = 'pending_review' ORDER BY random() LIMIT 5;` |
| 4 | Code | PrepareSlackMsg | Transforms products into Slack Block Kit |
| 5 | Slack | SendToSlack | Posts messages to `#queue-reviewer` |

---

## Slack Integration

### Slash Command Setup

**To configure the `/review-queue` slash command in Slack:**

1. Go to your Slack app settings (TechLoop Automation bot)
2. Navigate to: Features → Slash Commands
3. Create New Command:
   - **Command:** `/review-queue`
   - **Request URL:** `https://n8n.reganmcgregor.com.au/webhook/slack/review-queue`
   - **Short Description:** "Get 5 products from the review queue"
   - **Method:** `POST`
4. Reinstall your app if prompted

### Using the Slash Command

Type `/review-queue` in any Slack channel to instantly receive 5 random products for review. The bot will post individual HITL messages to `#queue-reviewer`.

### HITL Buttons

Each product message includes three action buttons:

1. **Approve** (Primary/Blue)
   - `action_id: approve_product`
   - Sets product status to 'approved'

2. **Reject** (Danger/Red)
   - `action_id: reject_product`
   - Prompts for rejection reason

3. **🔍 Google Search** (Link)
   - Opens Google search with product MPN or name
   - Perfect for mobile research
   - No interaction handler needed (external link)

**Button Value:** Queue item `id` (UUID from `tl_onboarding_queue`)

### Mobile Review Workflow

The Google Search button makes mobile review efficient:

1. Type `/review-queue` in Slack
2. Tap **🔍 Google Search** on any product
3. Research specs, images, pricing
4. Return to Slack → Tap **Approve** or **Reject**

---

## Related Documentation

- [[Workflow 5 - Product Detector]]
- [[Slack Integration]]
- [[Database Schema - Product Automation]]
