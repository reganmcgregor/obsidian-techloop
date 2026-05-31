# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with this vault.

## Overview

This is an **Obsidian vault** for **TechLoop**, an ecommerce store. This vault is used as a **Second Brain and Wiki** for business operations, product management, strategic planning, and infrastructure documentation.

**Important**:
- This is NOT a code repository — it's a business knowledge management vault.
- **Task Management**: All tasks are managed in **Notion** (per-department Tasks DBs), not in this vault. Do NOT add inline checkboxes (`- [ ]`) in markdown. When proposing or describing work, reference the relevant Notion task by URL or Task ID.
- **Role**: This vault serves as the central source of truth for knowledge, specifications, and planning.
- **Notion-as-OS**: The vault is organised by department and kept in sync with Notion via the `/publish` skill. Notion is the task/project management layer; Obsidian is the knowledge/spec layer.

## Obsidian Configuration

**Community plugin:** Obsidian Local REST API (enables external API access to vault).
(Standard built-in plugins active: Templates, Canvas, Graph view, Backlinks.)

## Working with This Vault

### Department-Based Organisation

The vault is organised by department (replacing the former PARA method). Each department folder contains three standard sub-folders:

| Sub-folder | Purpose |
|---|---|
| `Projects/` | Active project specs, briefs, and working documents |
| `Context/` | Reference material, brand guides, infrastructure docs |
| `Playbooks/` | Step-by-step operating procedures |

**Department folders:**

- `Departments/General/` — Cross-functional, company-wide knowledge
- `Departments/Marketing/` — Campaigns, brand, tone of voice, content drafts
- `Departments/Purchasing/` — Supplier sourcing, purchase orders, pricing
- `Departments/Engineering/` — Infrastructure, automation, technical specs
- `Departments/Customer-Service/` — Support procedures, FAQs, escalation guides
- `Departments/Operations/` — Fulfilment, logistics, internal processes

**Notable migrations from PARA:**
- Engineering project folders (`TechLoop Inventory Automation`, `TechLoop Shopify Migration`, `TechLoop Store Migration - Ecomus Theme`) → `Departments/Engineering/Projects/`
- `3-Resources/Infrastructure/` → `Departments/Engineering/Context/Infrastructure/`
- Tone of Voice + AI Writing Guidelines → `Departments/Marketing/Context/`

### Obsidian Features to Use

**Links:** Use `[[Note Title]]` to create links between notes.
**Tags:** Use `#tag` to categorise notes (e.g., `#product`, `#marketing`, `#infrastructure`).
**Canvas:** Create visual diagrams linking related notes and concepts.
**Templates:** Use templates for structured notes (Projects, Products).

### TechLoop Business Context

**Business Type:** Ecommerce retail store
**Key Business Areas:**
- Product sourcing and inventory management
- Online marketing and customer acquisition
- Order fulfilment and customer service
- Website optimisation and user experience

## Directory Structure

```
.
├── Departments/
│   ├── General/             # Cross-functional knowledge
│   │   ├── Projects/
│   │   ├── Context/
│   │   └── Playbooks/
│   ├── Marketing/           # Campaigns, brand, tone of voice
│   │   ├── Projects/
│   │   ├── Context/
│   │   ├── Playbooks/
│   │   └── Content-drafts/
│   ├── Purchasing/          # Suppliers, pricing, purchase orders
│   │   ├── Projects/
│   │   ├── Context/
│   │   └── Playbooks/
│   ├── Engineering/         # Infrastructure, automation, tech specs
│   │   ├── Projects/        # Incl. Inventory Automation, Shopify Migration
│   │   ├── Context/
│   │   │   └── Infrastructure/   # SQL schemas, Server docs
│   │   └── Playbooks/
│   ├── Customer-Service/    # Support, FAQs, escalation
│   │   ├── Projects/
│   │   ├── Context/
│   │   └── Playbooks/
│   └── Operations/          # Fulfilment, logistics, processes
│       ├── Projects/
│       ├── Context/
│       └── Playbooks/
├── _scratch/                # Raw analysis, agent working files (Obsidian-only)
├── _archive/                # Inactive / completed items (replaces 4-Archives/)
└── _templates/              # Obsidian templates
    ├── Project.md
    └── Product.md
```

## Editing Content
- **Read First:** Always read existing files before making changes.
- **Task Management:** Do NOT add task checkboxes (`- [ ]`). Tasks live in Notion — reference them by URL or Task ID.
- **Formatting:** Preserve markdown formatting and internal links.
- **Conventions:** Follow Australian English spelling (e.g., "organised", "optimise", "behaviour").
- **Templates:** Use `_templates/` when creating standard note types.
- **Shell paths:** Always double-quote paths in bash commands — the vault root contains spaces and a tilde (`iCloud~md~obsidian`). Example: `ls "<vault>/Departments/Engineering/"`.
- **Not a git repo:** This vault is not under version control. Don't run `git` operations in the vault root.

## Best Practices
- Keep technical specifications (SQL, n8n) in `Departments/Engineering/Context/Infrastructure/`.
- Use `Departments/<dept>/Projects/` for planning and spec work; track actual "To Do" items in Notion.
- Archive completed projects to `_archive/` to keep the vault clean.
- Use `_scratch/` for agent working files, raw analysis, and temporary notes — do not treat as permanent knowledge.

## Notion Integration

### Task Management — Notion (not Linear)
All tasks and project tracking live in **Notion**. Each department has its own teamspace with a Tasks database and a Projects database.

**Do NOT create `- [ ]` checkboxes in vault notes.** When referencing work to be done, link to the Notion task by URL or by Task ID (once `unique_id` properties are present).

### Notion Teamspace IDs
| Department | Teamspace UUID |
|---|---|
| General | `8ac37ca1-8c0b-40cf-900b-27cef2c7ca2d` |
| Marketing | `3558c3d3-c03e-8191-9af5-00426c0fca76` |
| Purchasing | `3558c3d3-c03e-81ee-adb8-004265de035d` |
| Engineering | `3558c3d3-c03e-81f4-808c-00427b3b52e1` |
| Customer Service | `3558c3d3-c03e-8171-8243-004243074b5d` |
| Operations | `3558c3d3-c03e-819d-9e11-0042de6c2c95` |

### Default Task Assignee
When creating Notion tasks on behalf of the user, default the **Assignee** property to:
- **User UUID:** `c2a5634f-3ff0-41cb-9cee-69bfb769f5b4` (Regan McGregor)

### Memory Files for Canonical IDs
Consult these memory files for up-to-date IDs rather than hardcoding:
- `<memory>/notion_user.md` — user UUID
- `<memory>/notion_teamspaces.md` — all 6 teamspace UUIDs
- `<memory>/notion_databases.md` — all DB IDs across teamspaces
- `<memory>/linear_migration.md` — TEC-N → Notion page ID map for the 15 Linear-imported tasks (Phase 3, May 2026)

### Project Specs Live in DB Row Bodies
Engineering project specs (Inventory Automation, Shopify Migration, Ecomus) live as the **body of the corresponding row** in the Projects (Engineering) DB — NOT as standalone pages. Updating a spec means editing the DB row's page body. The Obsidian markdown file's `notion_page_id` points at the DB row, not a standalone page.

### Notion Sync — `/publish` Skill
Notes that are synced to Notion carry YAML front matter:

```yaml
---
notion_page_id: <uuid or null>
notion_parent_id: <parent uuid>
notion_teamspace: general | marketing | purchasing | engineering | customer-service | operations
canonical: obsidian | notion
department: marketing
type: context | playbook | project-spec | reference
synced_at: <ISO 8601 UTC>
---
```

**Canonical rules:**
- `canonical: obsidian` — Obsidian is the source of truth; push content to Notion (e.g., engineering specs, project drafts).
- `canonical: notion` — Notion is the source of truth; pull content into Obsidian for agent filesystem reads (e.g., brand voice, tone of voice, competitor docs).

The `/publish` skill handles bidirectional sync based on these values.

### Schema YAML — Canonical Tasks/Projects Schema
The canonical property schema for all Tasks and Projects databases lives at:

```
Departments/General/Context/tasks-schema.yaml
```

This file is `canonical: obsidian` — Obsidian is the source of truth. The schema is published to the **Tasks Schema Reference** page in Notion (page ID in `<memory>/notion_databases.md`).

### `/sync-tasks-schema` Skill
Propagates the canonical schema from `tasks-schema.yaml` to all 12 dept Tasks + Projects DBs in Notion.

**Usage:**
- `/sync-tasks-schema` — sync all 12 DBs
- `/sync-tasks-schema --dry-run` — preview changes without writing
- `/sync-tasks-schema --db <slug>` — target a single DB (e.g., `--db marketing-tasks`)

**Limitations (require manual Notion UI work):**
- Status option names and colours — Notion API cannot rename or recolour existing status options
- Default Assignee — Notion API cannot set person-property defaults

### Phase 2 Dept-Specific Databases
Seven new databases were added in Phase 2. For DB IDs and Data Source IDs, consult `<memory>/notion_databases.md` (Phase 2 dept-specific DBs section).

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
Each department has a `Playbooks/` subfolder in Obsidian, mirrored to a `📚 Playbooks` parent page in the corresponding Notion teamspace. Playbook parent page IDs are in `<memory>/notion_databases.md`.

- **Engineering playbooks** — `canonical: obsidian` (Obsidian is source of truth; pushed to Notion)
- **All other dept playbooks** — `canonical: notion` (Notion is source of truth; pulled into Obsidian for agent reads)

19 playbook pages exist across all 6 departments. Phase 3 populated 6 of them with first-pass content:
- Engineering: Deploy Checklist, n8n Workflow Update Process, Schema Deployment, Incident Response
- General: Weekly Review, Monthly Retro

The remaining 13 playbook stubs live in Marketing / Purchasing / Customer-Service / Operations — fill ad-hoc when you actually run those workflows.

### Outstanding manual UI cleanup
See `Departments/General/Playbooks/Notion Cleanup.md` (also published to Notion as the **📋 Notion Cleanup** playbook in General Playbooks). Items remaining are UI-only and confirmed unreachable via API: status option renames, default Assignee templates, linked-DB views in dept hubs, optional Bin cleanup.