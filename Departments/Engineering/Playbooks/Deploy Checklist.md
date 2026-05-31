---
notion_page_id: 3558c3d3-c03e-81ea-b062-f35aa2d157c0
notion_parent_id: 3558c3d3-c03e-8120-ac7a-eda8804ac5ae
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: playbook
synced_at: 2026-05-03T16:37:50Z
---

# Deploy Checklist

> Pre/post checks for deploying TechLoop infrastructure changes.

## When to use

Any change to: n8n workflows, Supabase schema, server config, WooCommerce/WordPress plugins, or scrapers. Also use before enabling/disabling any schedule trigger.

## Steps

### Pre-deploy

1. Confirm the full scope of the change (which workflows, tables, or services are affected).
2. For schema changes: back up the affected table (`pg_dump` or export via Supabase Studio) before running DDL.
3. Check for active workflow runs — look at `tl_workflow_executions` for `status = 'running'`; wait for them to complete.
4. Confirm all required credentials are available (Postgres `BSoGuZ9BOv4OWqXf`, WooCommerce `vMZW4D7gnzSYVw0g`, Ollama `mLc8L4ENefXAw6xJ`, Slack `DvtYgxYgI1p5Q3QX`).
5. For scheduled workflows: note the workflow ID — you will need to deactivate+reactivate after the update.

### Deploy

6. Apply the change (n8n partial update, Supabase migration, server config edit, etc.).
7. For n8n partial updates on scheduled workflows — immediately run the deactivate/reactivate sequence:
   ```bash
   curl -X POST http://localhost:5678/api/v1/workflows/{id}/deactivate -H 'X-N8N-API-KEY: KEY'
   curl -X POST http://localhost:5678/api/v1/workflows/{id}/activate   -H 'X-N8N-API-KEY: KEY'
   ```
   (API key: `docker exec n8n_postgres_db psql -U n8n_user -d n8n -c "SELECT \"apiKey\", label FROM user_api_keys LIMIT 5;"`)
8. Run the workflow once manually (if webhook/manual trigger) or fire a test payload to verify it executes without error.

### Post-deploy

9. Monitor the next 1–2 scheduled runs via `tl_workflow_executions` and check `tl_sync_log` for errors.
10. Confirm Slack notifications fire correctly (check for `no_text` errors in n8n execution logs).
11. Update the relevant Obsidian doc (`3-Resources/Infrastructure/` or `1-Projects/`) to reflect the change.

## Notes / gotchas

- Schedule Triggers must be **deactivated then reactivated** after any `n8n_update_partial_workflow` call — the trigger silently deregisters without this step.
- The Schedule Trigger test API does not work — only webhook/manual/form triggers can be tested via the API.
- Edit Image nodes: always re-include `operation: "border"` explicitly — it gets dropped on every `updateNode`.
- `n8n_update_full_workflow` replaces ALL node parameters — avoid unless doing a full rebuild. Prefer partial updates.
- WooCommerce API keys are domain-specific (`www.techloop.com.au` is canonical).

## Last reviewed

2026-05-03 by Regan
