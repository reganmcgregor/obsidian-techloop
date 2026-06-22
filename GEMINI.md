# GEMINI.md

This file provides context and instructions for Gemini when working within the **TechLoop Obsidian Vault**.

## 1. Vault Overview

**TechLoop** is an Australian ecommerce store specializing in computers, gaming, and electronics. This Obsidian vault serves as the **Second Brain and Wiki** for the business.

*   **Location:** `/Users/regan.mcgregor/Library/Mobile Documents/iCloud~md~obsidian/Documents/TechLoop`
*   **Purpose:** Central knowledge base, project specifications, and infrastructure documentation.
*   **Task Management:** All tasks are tracked in **Notion** (per-department Tasks DBs). This vault is for *knowledge*, not *task tracking*. Do NOT create `- [ ]` checkboxes — reference Notion tasks by URL or Task ID instead.
*   **Notion-as-OS:** Notion is the task/project management layer; Obsidian is the knowledge/spec layer. Bidirectional sync is managed via the `/publish` skill and YAML front matter.

### Directory Structure (Departments)
*   `Departments/General/`: Cross-functional knowledge — Projects, Context, Playbooks.
*   `Departments/Marketing/`: Campaigns, brand, tone of voice, content drafts.
*   `Departments/Purchasing/`: Suppliers, pricing, purchase orders.
*   `Departments/Engineering/`: Infrastructure, automation, technical specs (incl. `Context/Infrastructure/`).
*   `Departments/Customer-Service/`: Support procedures, FAQs, escalation guides.
*   `Departments/Operations/`: Fulfilment, logistics, internal processes.
*   `_scratch/`: Agent working files and raw analysis (Obsidian-only, not synced to Notion).
*   `_archive/`: Inactive/completed items (replaces `4-Archives/`).
*   `_templates/`: Templates for standard notes (Product, Project).

**Notable migrations from PARA:**
*   Engineering project folders (`TechLoop Inventory Automation`, `TechLoop Shopify Migration`, `TechLoop Store Migration - Ecomus Theme`) → `Departments/Engineering/Projects/`
*   `3-Resources/Infrastructure/` → `Departments/Engineering/Context/Infrastructure/`
*   Tone of Voice + AI Writing Guidelines → `Departments/Marketing/Context/`

---

## 2. Active Focus: Inventory Automation

The primary active engineering project is the **Inventory Automation** system.

*   **Master Note:** `Departments/Engineering/Projects/TechLoop Inventory Automation/Master Note - Inventory Automation.md`
*   **Status:** Phase 1 (Foundation) Complete. Phase 1.5 (ACF Migration) Ready to Execute.
*   **Task Tracking:** Notion — Engineering teamspace Tasks DB (UUID: `3558c3d3-c03e-81f4-808c-00427b3b52e1`)

### System Architecture
*   **Source:** Leader (Supplier) XML Feed.
*   **Middleware:** n8n (Workflows for Ingest, Mirror, Sync).
*   **Database:** Supabase PostgreSQL (5 tables, 3 views).
*   **Destination:** WooCommerce.

### Key Resources
*   **SQL Schema:** `Departments/Engineering/Context/Infrastructure/Database Schema - TechLoop Inventory.sql`
*   **Server Config:** `Departments/Engineering/Context/Infrastructure/Ubuntu Server - 192.168.1.111.md`
*   **n8n Utilities:** `Departments/Engineering/Context/Infrastructure/n8n-utilities.js`

---

## 3. Conventions & Guidelines

### File Operations
*   **Read First:** Always check the relevant project note or Context doc before making changes.
*   **Internal Links:** Use `[[Note Name]]` to maintain the knowledge graph.
*   **No Tasks:** Do not create `- [ ]` task lists. Tasks live in Notion — reference by URL or Task ID.
*   **Australian English:** Use British/Australian spelling (e.g., "organised", "optimisation", "behaviour").

### Infrastructure Context
*   **Server:** Ubuntu 25.04 @ `192.168.1.111`
*   **Supabase:** `https://supabase.labgregor.dev`
*   **n8n:** `https://workflows.labgregor.dev`

### Engineering Focus
*   `Departments/Engineering/Projects/` contains *technical specifications* and *logic* for engineering tasks. The execution steps are tracked in Notion (Engineering teamspace). Use these notes to understand *how* to build the solution.

---

## 4. Notion Integration

### Teamspace IDs
| Department | Teamspace UUID |
|---|---|
| General | `8ac37ca1-8c0b-40cf-900b-27cef2c7ca2d` |
| Marketing | `3558c3d3-c03e-8191-9af5-00426c0fca76` |
| Purchasing | `3558c3d3-c03e-81ee-adb8-004265de035d` |
| Engineering | `3558c3d3-c03e-81f4-808c-00427b3b52e1` |
| Customer Service | `3558c3d3-c03e-8171-8243-004243074b5d` |
| Operations | `3558c3d3-c03e-819d-9e11-0042de6c2c95` |

### Default Task Assignee
When creating Notion tasks, default the **Assignee** to:
*   **User UUID:** `c2a5634f-3ff0-41cb-9cee-69bfb769f5b4` (Regan McGregor — `regan@techloop.com.au`)

### Memory Files for Canonical IDs
*   `<memory>/notion_user.md` — user UUID
*   `<memory>/notion_teamspaces.md` — all 6 teamspace UUIDs
*   `<memory>/notion_databases.md` — all DB IDs across teamspaces
*   `<memory>/linear_migration.md` — TEC-N → Notion page ID map for the 15 Linear-imported tasks (Phase 3, May 2026)

### Project Specs Live in DB Row Bodies
Engineering project specs (Inventory Automation, Shopify Migration, Ecomus) live as the **body of the corresponding row** in the Projects (Engineering) DB — NOT as standalone pages. Updating a spec means editing the DB row's page body. The Obsidian markdown file's `notion_page_id` points at the DB row, not a standalone page.

### Notion Sync — `/publish` Skill
Notes synced to Notion carry YAML front matter. Key fields:

```yaml
---
notion_page_id: <uuid or null>
notion_parent_id: <parent uuid>
notion_teamspace: general | marketing | purchasing | engineering | customer-service | operations
canonical: obsidian | notion
department: engineering
type: context | playbook | project-spec | reference
synced_at: <ISO 8601 UTC>
---
```

*   `canonical: obsidian` → Obsidian wins; push to Notion (e.g., engineering specs, project drafts).
*   `canonical: notion` → Notion wins; pull into Obsidian for agent reads (e.g., brand voice, tone, competitor docs).

### Schema YAML — Canonical Tasks/Projects Schema
The canonical property schema for all Tasks and Projects databases:

```
Departments/General/Context/tasks-schema.yaml
```

This file is `canonical: obsidian`. It is published to the **Tasks Schema Reference** page in Notion (page ID in `<memory>/notion_databases.md`).

### `/sync-tasks-schema` Skill
Propagates `tasks-schema.yaml` to all 12 dept Tasks + Projects DBs in Notion.

*   `/sync-tasks-schema` — sync all 12 DBs
*   `/sync-tasks-schema --dry-run` — preview changes without writing
*   `/sync-tasks-schema --db <slug>` — target a single DB (e.g., `--db purchasing-tasks`)

**Limitations requiring manual Notion UI:** status option names/colours and default Assignee values cannot be set via the Notion API.

### Phase 2 Dept-Specific Databases
Seven new databases were added in Phase 2. For DB IDs, consult `<memory>/notion_databases.md` (Phase 2 section).

| DB | Teamspace |
|---|---|
| OKRs | General |
| Decisions Log | General |
| Content Calendar | Marketing |
| Social Posts | Marketing |
| Email/Newsletter | Marketing |
| Articles | Customer Service |
| Returns | Operations |

### Playbook Conventions
Each department has `Departments/<dept>/Playbooks/` in Obsidian, mirrored to a `📚 Playbooks` parent page in the corresponding Notion teamspace. Playbook parent page IDs are in `<memory>/notion_databases.md`.

*   Engineering playbooks — `canonical: obsidian` (pushed to Notion)
*   All other dept playbooks — `canonical: notion` (pulled into Obsidian for agent reads)

19 playbook pages exist across all 6 departments. Phase 3 populated 6 of them with first-pass content:
*   Engineering: Deploy Checklist, n8n Workflow Update Process, Schema Deployment, Incident Response
*   General: Weekly Review, Monthly Retro

The remaining 13 playbook stubs live in Marketing / Purchasing / Customer-Service / Operations — fill ad-hoc when you actually run those workflows.

### Outstanding manual UI cleanup
See `Departments/General/Playbooks/Notion Cleanup.md` (also published to Notion as the **📋 Notion Cleanup** playbook in General Playbooks). Items remaining are UI-only and confirmed unreachable via API: status option renames, default Assignee templates, linked-DB views in dept hubs, optional Bin cleanup.
