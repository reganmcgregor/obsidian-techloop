---
notion_page_id: 3558c3d3-c03e-81da-a5dd-d87ecd410548
notion_parent_id: 3558c3d3-c03e-81f6-a928-d6d61e36ec51
notion_teamspace: general
canonical: obsidian
department: general
type: playbook
synced_at: 2026-05-03T16:48:29Z
---

# Notion Cleanup Checklist

> One-stop list of manual UI work in Notion that the API can't do for us. Knock these out in a single sitting (~30-45 min) to get the workspace fully aligned.

## When to use

After running Phase 1, 2, or 3 of the Notion-as-OS automation. Items here are confirmed UI-only (Notion MCP can't update them).

## Items

### 1. Status options on all 12 dept Tasks/Projects DBs

The Notion API cannot edit status property options. Each dept DB was created with Notion's defaults (`Not started` / `In progress` / `Done`). The canonical YAML schema (`Departments/General/Context/tasks-schema.yaml`) defines the actual options.

**Tasks DBs — apply these status options on each:**
- to_do group: `Not Started`
- in_progress group: `In Progress`, `Blocked`
- complete group: `Done`, `Archived`

For each Tasks DB (General, Marketing, Purchasing, Engineering, Customer Service, Operations):
- Open DB → click Status property header → Edit property → groups view
- Rename Notion's default options to match (e.g. "Not started" → "Not Started" with capital S)
- Add `Blocked` to in_progress group
- Add `Archived` to complete group

**Projects DBs — apply these:**
- to_do group: `Backlog`, `Planning`
- in_progress group: `In Progress`, `Paused`
- complete group: `Done`, `Canceled`

For each Projects DB (General, Marketing, Purchasing, Engineering, Customer Service, Operations): rename and add as above.

### 2. Default Assignee on all 6 Tasks DBs

The DB default page template can't be set via MCP.

For each Tasks DB:
- Open DB → click `⋯` (more options) → "Edit default properties" (or click "+" beside the default template)
- Set `Assignee` default to **Regan McGregor** (UUID `c2a5634f-3ff0-41cb-9cee-69bfb769f5b4`)

### 3. Customer Feedback DB Status options

Currently has Notion defaults. Rename to match the canonical from Phase 1 spec:
- to_do: `New`
- in_progress: `In Review`, `Investigating`
- complete: `Actioned`, `Won't Do`, `Resolved`

### 4. Drag pages into General teamspace (if still pending)

Phase 1 created two pages at workspace root because the API rejects teamspace IDs as page parents for some operations:

- `📚 Reference Databases` (holder for Suppliers, Brands, Competitors, Customer Feedback DBs) — drag into General teamspace
- `📋 Tasks Schema Reference` page (you may have already moved this) — should sit under General

In Notion sidebar: locate at workspace root → drag onto General teamspace.

### 5. Linked-DB views in dept hubs (cross-teamspace)

Phase 1+2 created hub pages with placeholders for "Cross-cutting tasks linked to this dept's projects" and similar. These linked views **must be created in the UI** because the MCP cannot embed a linked database from a different teamspace.

For each dept hub (Marketing, Purchasing, Engineering, Customer Service, Operations):
- Open the hub page
- In the "Cross-cutting tasks linked to a [Dept] project" section: type `/linked` → "Linked database view" → select General Tasks DB → filter where `Project (General)` contains a [Dept] project
- Save the view

For Purchasing hub specifically: also add linked views of Suppliers DB and Brands DB (filter as desired).
For CS hub specifically: add a linked view of Customer Feedback DB sorted by Frequency desc.
For Home page (in General): add 6 linked views (one per dept's Projects DB filtered to Status = In Progress).

### 6. Operations Projects: dates schema decision

Projects (Operations) has separate `Start Date` + `Target Date` properties instead of the canonical single `Dates` (range). Pick one:
- **Keep both** — update `tasks-schema.yaml`'s projects.canonical to use start/target instead of date range.
- **Consolidate** — manually merge the two into a single `Dates` range property, delete the old ones.

The first option is less work. Edit the YAML accordingly so `/sync-tasks-schema --dry-run` returns clean.

### 7. Priority extras decision (CS + Ops)

Tasks (CS), Tasks (Ops), Projects (CS), Projects (Ops) have extra Priority options (`Urgent` and `Critical`) beyond the canonical Low/Medium/High. Pick:
- **Keep extras** — update `tasks-schema.yaml`'s priority options to include Urgent / Critical.
- **Remove extras** — manually delete those options from the 4 DBs (they may have been used in real tasks; check first).

### 8. Reference DB population (Suppliers, Brands, Competitors, Customer Feedback)

These DBs exist but are empty. Populate as you have data:
- **Suppliers** — start with active supplier list (Leader, etc.)
- **Brands** — extract from product catalogue
- **Competitors** — start with 5-10 known competitors with URLs
- **Customer Feedback** — start adding themes from HubSpot tickets ad-hoc

Not blocking — populate over time.

### 9. (Optional) Move 22 retroactive tasks into a Phase 6 sub-page

20 retroactive Done tasks were created flat in the Tasks (Engineering) DB during Phase 3. If you want them grouped under a "Phase 6" Project record, create a Projects (Engineering) row titled "Phase 6: Product Onboarding & Enrichment" and re-link each retroactive task to it.

### 10. Verify Linear migration, then archive Linear team

15 Linear issues are mirrored in Notion (Tasks (Engineering) with `Source: Linear-imported`, `Linear ID: TEC-N`). Once you've spot-checked a few:
- Archive the Linear `TechLoop` team to remove it from active rotation.
- (Or just stop opening Linear — it's all in Notion now.)

## Notes / gotchas

- Status options + default Assignee are the most disruptive items left — get those done first.
- Linked DB views are cosmetic but make hub pages much more useful.
- Don't delete anything in Linear until you've verified Notion is the source of truth for at least a week.

## Last reviewed

2026-05-04 by Regan
