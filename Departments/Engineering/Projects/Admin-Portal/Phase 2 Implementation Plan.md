# Phase 2 Implementation Plan

This plan details the implementation steps for Phase 2 of the TechLoop Admin Portal. Phase 2 focuses on improving the queue workflow, category management, handling mistakes, and integrating the publisher workflow.

## User Review Required

> [!IMPORTANT]
> Please review the open questions below to ensure the implementation matches your workflow preferences. Once approved, I will begin execution and also mirror this finalized plan to your Obsidian Vault.

## Open Questions

> [!WARNING]
> 1. **Default Category Mappings:** Should unmapped categories default to a specific "Review Required" status, or should they default to `is_blocked = true` to prevent accidental publishing of new/unknown categories?
> 2. **Reverting Products:** When a product is "Reverted to Pending" from the History tab, should we automatically clear any metadata that was modified during its previous review, or keep the modified metadata but reset its status?

## Proposed Changes

### 1. Queue Upgrades

Enhance the main queue interface for better review efficiency.

#### [MODIFY] [src/app/queue/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/queue/page.tsx)
- Add Category and Brand filters to the queue view.
- Add an Image Toggle to hide/show product images for faster scanning.

#### [MODIFY] [src/app/queue/data-table.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/queue/data-table.tsx)
- Expand rows to display Product Detector metadata (dimensions, weight, raw data) inline.

---

### 2. Category Mapping

Build a dedicated interface for managing category rules and mappings to WooCommerce (WC).

#### [NEW] [src/app/categories/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/categories/page.tsx)
- Implement a data table for Categories.
- Add toggles for `is_blocked` and `auto_approve`.
- Add mapping fields for WooCommerce category IDs/names.
- Include bulk update capabilities.

---

### 3. History/Mistakes View

Provide a safety net for accidental approvals or rejections.

#### [NEW] [src/app/history/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/history/page.tsx)
- Create a view displaying recently accepted and rejected items.
- Implement a "Revert to Pending" action to reset product status and move it back to the queue.

#### [NEW] [src/app/history/actions.ts](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/history/actions.ts)
- Add server actions for fetching historical items and reverting their status in Supabase.

---

### 4. Publisher Workflow

Monitor and manage the downstream n8n publisher.

#### [NEW] [src/app/publishing/page.tsx](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/publishing/page.tsx)
- Create a tab watching the n8n publisher's status.
- Display products with statuses: `processing`, `published`, and `failed`.
- Show error logs for failed publications.
- Implement a "Retry" button to resend failed items to n8n.

#### [NEW] [src/app/publishing/actions.ts](file:///Users/regan.mcgregor/Sites/techloop-admin-portal/src/app/publishing/actions.ts)
- Add server actions to fetch publishing status from Supabase (or n8n) and trigger retries.

## Verification Plan

### Automated Tests
- Run `npm run build` to ensure no TypeScript or build errors are introduced.
- Run Next.js linting `npm run lint`.

### Manual Verification
- **Queue:** Verify filters work and row expansion correctly displays metadata.
- **Categories:** Update a category mapping and verify it saves to the database.
- **History:** Approve an item, navigate to History, and click "Revert to Pending". Verify it appears back in the queue.
- **Publishing:** Trigger a retry for a failed item and ensure the status updates to processing.
