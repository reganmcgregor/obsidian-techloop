---
notion_page_id: 3558c3d3-c03e-818a-8c00-ec68e6a6cfc1
notion_parent_id: 3558c3d3-c03e-8120-ac7a-eda8804ac5ae
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: playbook
synced_at: 2026-05-03T16:38:28Z
---

# n8n Workflow Update Process

> Safe process for editing live n8n workflows (especially scheduled ones).

## When to use

Any change to a deployed n8n workflow — adding/removing nodes, updating parameters, changing credentials, or restructuring connections.

## Steps

1. **Identify the workflow ID** via `n8n_list_workflows` or the vault docs under `1-Projects/TechLoop Inventory Automation/`.
2. **Snapshot first** — run `n8n_get_workflow` and save the JSON locally if the change is risky or involves removing nodes.
3. **Prefer `n8n_update_partial_workflow`** over `n8n_update_full_workflow`. Full updates silently drop parameters that aren't included.
4. For `updateNode`: pass **complete** `parameters` AND `credentials` objects — the call replaces both entirely. Including only the changed field drops everything else.
5. Apply the change.
6. **If the workflow has a Schedule Trigger** — deactivate then reactivate immediately:
   ```bash
   curl -X POST http://localhost:5678/api/v1/workflows/{id}/deactivate -H 'X-N8N-API-KEY: KEY'
   curl -X POST http://localhost:5678/api/v1/workflows/{id}/activate   -H 'X-N8N-API-KEY: KEY'
   ```
7. **After `removeNode`** — verify SplitInBatches output connections. Removing a node at output[0] compacts indices: output[1] shifts to output[0]. Fix with removeNode+addNode on the loop body node if needed.
8. **Test** — run once manually if webhook/manual trigger. For Schedule workflows, trigger via a test data flow (the test API does not fire Schedule triggers).

## Notes / gotchas

| Issue | Correct approach |
|-------|-----------------|
| Referencing nodes by name | Use `nodeId` — the `name` field silently becomes empty string |
| AI sub-node connections | `sourceOutput: "ai_embedding"` (string, not number) |
| IF node branches | `branch: "true"` / `branch: "false"` |
| SplitInBatches v3 outputs | `main[0]` = done (fires once), `main[1]` = loop (fires per item) |
| HTTP Request JSON body | `specifyBody: "json"` + `jsonBody: "={{ JSON.stringify({...}) }}"` |
| `continueOnFail` | Deprecated — use `onError: "continueRegularOutput"` |
| HTTP from Code nodes | `$helpers`, `fetch`, and `require('http')` are unavailable — use an HTTP Request node |
| Object spread in expressions | `{ ...obj }` causes 500 errors — use `JSON.parse(JSON.stringify(obj))` to deep-clone |
| Slack block messages | Wrap in object: `blocksUi: "={{ { \"blocks\": $json.blocks } }}"` + `text` fallback param |
| IF node `isNotEmpty` | Use `$input.all().length > 0` (number > gt) — `isNotEmpty` with `singleValue: true` routes all to false |

**Never:** delegate `n8n_update_full_workflow` to subagents — too easy to silently lose node parameters.

## Last reviewed

2026-05-03 by Regan
