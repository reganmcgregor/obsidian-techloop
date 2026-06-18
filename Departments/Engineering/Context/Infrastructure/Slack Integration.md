## Overview

All TechLoop automation workflows send notifications to Slack instead of email. A dedicated **TL_Slack_Interaction_Handler** workflow processes button callbacks for Human-in-the-Loop (HITL) approval flows.

## Slack App

- **App Name:** TechLoop Automations
- **n8n Credential:** `TechLoop Automation` (ID: `DvtYgxYgI1p5Q3QX`, type: `slackApi` bot token)
- **Required Scopes:** `chat:write`, `chat:write.public`
- **Interactivity Request URL:** `https://workflows.labgregor.dev/webhook/slack-interaction`

## Channels

| Channel | Workflow | Purpose |
|---|---|---|
| `#leader-feed` | TL_Ingest_Leader_Feed | Feed ingest results (product count, duration, errors) |
| `#product-detector` | TL_Product_Detector | New products detected + HITL approve/reject buttons |
| `#inventory-sync` | TL_Inventory_Syncer | Sync stats (stock/cost changes, supplier status) |
| `#wc-mirror` | TL_Mirror_WooCommerce | Mirror sync results (mode, success/failure counts) |
| `#price-watchdog` | TL_Price_Watchdog | Low-margin product alerts (critical/warning) |
| `#product-publisher` | TL_Product_Publisher | Published products + HITL publish/edit/preview buttons |
| `#enrichment-review` | TL_Attribute_Proposer, TL_Enrichment_Reviewer | Attribute/term proposals + enrichment HITL review |

## HITL Buttons

### Product Detector (Approve/Reject)

When `pending_review` products are detected, individual Slack messages are sent to `#product-detector` with:
- **Approve** button (`action_id: approve_product`, value: queue ID) — sets `tl_onboarding_queue.status = 'approved'`
- **Reject** button (`action_id: reject_product`, value: queue ID) — sets `tl_onboarding_queue.status = 'rejected'`

### Product Publisher (Publish/Edit/Preview)

After a WooCommerce draft is created, a Slack message is sent to `#product-publisher` with:
- **Publish Now** button (`action_id: wc_publish_product`, value: WC product ID) — sets WC product status from `draft` to `publish`
- **Edit in WordPress** — link button opening the WP admin editor
- **Preview** — link button opening the product permalink

### Attribute Proposer (New Attributes/Terms — Phase 6.3)

When the LLM extracts attributes not yet in WooCommerce, `TL_Attribute_Proposer` groups them and sends batch messages to `#enrichment-review`:

- **Approve New Attribute** (`action_id: approve_new_attribute`, value: JSON `{proposal_id, attribute_name}`) — creates attribute in WC via `POST /products/attributes`, updates `tl_proposed_attributes.status = 'approved'`
- **Reject New Attribute** (`action_id: reject_new_attribute`, value: JSON `{proposal_id}`) — sets `tl_proposed_attributes.status = 'rejected'`
- **Map to Existing** (`action_id: map_to_existing`, type: `static_select` dropdown, value: JSON `{proposal_id, target_attribute_id, target_attribute_name}`) — inserts alias into `tl_attribute_aliases`, sets `tl_proposed_attributes.status = 'mapped'`. The dropdown lists all existing WC attributes so the reviewer can map a proposed attribute (e.g., "Memory") to an existing one (e.g., "RAM") instead of creating a duplicate.
- **Approve New Term** (`action_id: approve_new_term`, value: JSON `{proposal_id, wc_attribute_id, term_value}`) — creates term in WC via `POST /products/attributes/<id>/terms`, updates status
- **Reject New Term** (`action_id: reject_new_term`, value: JSON `{proposal_id}`) — sets `tl_proposed_attributes.status = 'rejected'`

### Enrichment Reviewer (Attribute Mappings — Phase 6.3)

`TL_Enrichment_Reviewer` sends per-product attribute mapping batches to `#enrichment-review` for approval:

- **Approve Mapping** (`action_id: approve_attr_mapping`, value: JSON `{mapping_id}`) — sets `tl_product_attributes.review_status = 'approved'`
- **Reject Mapping** (`action_id: reject_attr_mapping`, value: JSON `{mapping_id}`) — sets `tl_product_attributes.review_status = 'rejected'`

## TL_Slack_Interaction_Handler

- **Workflow ID:** `rDloLTYACT56kfpn`
- **Webhook Path:** `/webhook/slack-interaction`
- **Trigger:** Slack sends `POST application/x-www-form-urlencoded` with `payload` JSON field
- **Response:** Immediately returns 200 (Slack requires response within 3 seconds)
- **Routing:** Switch node routes by `action_id` to 10 branches: product approve/reject, WC publish, attribute approve/reject/map-to-existing, term approve/reject, mapping approve/reject
- **Value Formats:** Product actions use simple UUID/integer values; enrichment actions use JSON strings with `proposal_id` and metadata. The `map_to_existing` action uses a `static_select` dropdown (value from `selected_option.value` rather than `action.value`).
- **Confirmation:** Each branch updates the original Slack message via `response_url` to show who actioned it and when

## Block Kit Message Format

All workflow notifications use Slack Block Kit with a consistent structure:
1. **Header block** — workflow name with status emoji
2. **Section block** — key stats in mrkdwn format
3. **Context block** — timestamp in AEST
4. **Error section** (conditional) — only shown when errors occur

## Migration Notes

- Migrated from Elastic Email (SMTP) on 2026-02-07
- Previous email credential: `Elastic Email` (ID: `vkCbG62NopBnWPHl`) — can be deactivated
- Email nodes (`Prepare Email` + `Send Email`) were removed from all 6 workflows
