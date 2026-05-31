# AGENTS.md - TechLoop Knowledge Base & Automation Codebase

This file provides comprehensive guidance for agentic coding agents working with the TechLoop Obsidian vault and associated automation code.

## Project Overview

**TechLoop** is an ecommerce retail store using this Obsidian vault as a Second Brain and knowledge management system. The vault contains:
- Business operations documentation
- Product management and inventory specs
- Infrastructure documentation
- Automation workflows and utilities

**Key Context:**
- Primary focus is knowledge management, not traditional software development
- Task management is handled in **Notion** (per-department Tasks DBs) — not Linear, not GitHub issues
- Code components are utilities for n8n automation workflows
- The vault is organised by department under `Departments/`; Notion is the task/project layer; Obsidian is the knowledge/spec layer

## Build/Lint/Test Commands

### JavaScript Code (n8n Utilities)

Since this is primarily a documentation vault with utility scripts, traditional build processes are minimal. However, for the JavaScript utilities:

#### Testing Single Functions
```bash
# Test n8n utilities in Node.js environment
node -e "
const { toSnakeCase, sanitizeAvailability } = require('./Departments/Engineering/Context/Infrastructure/n8n-utilities.js');
// Uncomment the module.exports in n8n-utilities.js first
console.log('Testing toSnakeCase:', toSnakeCase('ProductName2'));
console.log('Testing sanitizeAvailability:', sanitizeAvailability('42'));
"
```

#### Basic Syntax Validation
```bash
# Check JavaScript syntax
node --check Departments/Engineering/Context/Infrastructure/n8n-utilities.js
```

#### Manual Testing Commands
```bash
# Test specific utility functions
node -e "
// Copy function from n8n-utilities.js and test here
function sanitizeAvailability(value) {
    if (value === null || value === undefined || value === '') return 0;
    const strValue = String(value).toUpperCase().trim();
    const unavailableKeywords = ['CALL', 'POA', 'N/A', 'TBA', 'TBC', 'UNKNOWN'];
    if (unavailableKeywords.includes(strValue)) return 0;
    const parsed = parseInt(value, 10);
    if (isNaN(parsed) || parsed < 0) return 0;
    return parsed;
}
console.log('Test 42:', sanitizeAvailability('42'));
console.log('Test CALL:', sanitizeAvailability('CALL'));
"
```

### Database Schema (SQL)

#### Schema Validation
```bash
# Validate SQL syntax (if PostgreSQL client available)
psql -f "Departments/Engineering/Context/Infrastructure/Database Schema - TechLoop Inventory.sql" --syntax-only
```

#### Deployment Verification
```sql
-- Verify table creation
SELECT table_name FROM information_schema.tables WHERE table_schema = 'public' AND table_name LIKE 'tl_%';

-- Check indexes
SELECT indexname FROM pg_indexes WHERE tablename LIKE 'tl_%';
```

## Code Style Guidelines

### JavaScript (n8n Utilities)

#### Function Structure
- **JSDoc Comments**: Every function must have comprehensive JSDoc documentation
- **Parameter Validation**: Always check for null/undefined inputs
- **Error Handling**: Use defensive programming - return safe defaults for invalid inputs
- **Philosophy**: "Raw over Modelled" - preserve original data structure when sanitizing

#### Naming Conventions
```javascript
// Functions: camelCase
function toSnakeCase(str) { ... }
function sanitizeAvailability(value) { ... }

// Variables: camelCase
const availabilityTotal = sanitizeAvailability(rawValue);

// Constants: UPPER_SNAKE_CASE
const UNAVAILABLE_KEYWORDS = ['CALL', 'POA', 'N/A'];
```

#### Code Organization
```javascript
// ============================================================================
// SECTION HEADER - Clear section demarcation
// ============================================================================

/**
 * Function description with:
 * - Purpose
 * - Parameters with types
 * - Return type
 * - Usage examples
 */
function functionName(param) {
    // Input validation first
    if (!param || typeof param !== 'expectedType') return defaultValue;

    // Main logic
    // ...

    // Return result
    return result;
}
```

#### Specific Patterns Used

**Data Sanitization Functions:**
- Always handle null/undefined/empty inputs
- Return safe defaults (0 for numbers, null for objects, false for booleans)
- Preserve original values when possible (e.g., `availability_total_raw`)

**Array/Object Processing:**
```javascript
// Safe array iteration
if (!Array.isArray(array)) return [];

// Safe object property access
const value = obj?.property ?? defaultValue;
```

### SQL Schema Conventions

#### Table Naming
- Prefix all tables with `tl_` (TechLoop)
- Use snake_case for all identifiers
- Example: `tl_feed_leader_raw`, `tl_sync_log`

#### Column Naming
- snake_case for all columns
- Append `_at` for timestamps: `created_at`, `updated_at`, `last_sync_at`
- Append `_raw` for preserved original values: `availability_total_raw`

#### Indexing Strategy
```sql
-- Primary keys: SERIAL
id SERIAL PRIMARY KEY,

-- Foreign keys: explicit naming
supplier_id INTEGER REFERENCES tl_suppliers(id),

-- Performance indexes on commonly queried columns
CREATE INDEX idx_table_column ON table_name(column_name);
```

#### Documentation
```sql
-- =====================================================================
-- SECTION HEADER - Clear demarcation
-- =====================================================================
-- Purpose: Brief description
-- Created: YYYY-MM-DD
-- =====================================================================
```

### Markdown Documentation

#### File Headers
```markdown
# Document Title

Brief description of document purpose and scope.

## Section Header

Content with proper formatting and internal links.
```

#### Code Blocks
- Use language-specific syntax highlighting
- Include executable examples when possible
- Comment code examples clearly

#### Australian English
- Use British/Australian spelling: `organisation`, `optimisation`, `behaviour`
- Follow local date formats: `DD/MM/YYYY`

## File Organization

### Directory Structure
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
│   │   │   └── Infrastructure/   # SQL schemas, Server docs, n8n-utilities.js
│   │   └── Playbooks/
│   ├── Customer-Service/    # Support, FAQs, escalation
│   │   ├── Projects/
│   │   ├── Context/
│   │   └── Playbooks/
│   └── Operations/          # Fulfilment, logistics, processes
│       ├── Projects/
│       ├── Context/
│       └── Playbooks/
├── _scratch/                # Agent working files, raw analysis (Obsidian-only)
├── _archive/                # Inactive / completed items
├── _templates/              # Obsidian templates
└── AGENTS.md                # This file
```

### File Naming
- Use descriptive, hyphen-separated names
- Include dates for versioned documents: `Database Schema - TechLoop Inventory.sql`
- Use consistent prefixes for related files

## Development Workflow

### For Code Changes
1. **Read existing code** first to understand patterns
2. **Follow established conventions** from n8n-utilities.js
3. **Add comprehensive documentation** for any new functions
4. **Test manually** using Node.js console commands
5. **Preserve data integrity** - follow "Raw over Modelled" philosophy

### For Documentation Changes
1. **Use existing templates** from `_templates/` directory
2. **Follow department-based organisation** — place notes under `Departments/<dept>/{Projects,Context,Playbooks}/`
3. **Maintain internal links** with `[[Note Title]]` format
4. **Use consistent formatting** and Australian English
5. **Do NOT add task checkboxes** — tasks managed in Notion (per-department Tasks DBs)

## Quality Assurance

### Code Review Checklist
- [ ] JSDoc documentation complete for all functions
- [ ] Input validation implemented
- [ ] Error handling with safe defaults
- [ ] Naming conventions followed
- [ ] Code organization matches existing patterns
- [ ] Manual testing performed

### Documentation Review Checklist
- [ ] Proper markdown formatting
- [ ] Internal links working
- [ ] Australian English spelling
- [ ] Consistent with existing templates
- [ ] No task checkboxes added

## Tools & Dependencies

### Required for Development
- **Node.js**: For JavaScript utility testing
- **PostgreSQL client**: For SQL validation
- **Obsidian**: For documentation editing

### No Build System
- This is not a traditional application with build pipelines
- Code is primarily utility functions for n8n workflows
- Documentation is maintained in Obsidian vault format

## Notion Integration

### Task Management — Notion (not Linear)
All tasks and project tracking live in **Notion**. Each department has its own teamspace with a Tasks DB and Projects DB.

**Do NOT create `- [ ]` checkboxes in vault notes.** Reference work by Notion task URL or Task ID.

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
When creating Notion tasks, default the **Assignee** to:
- **User UUID:** `c2a5634f-3ff0-41cb-9cee-69bfb769f5b4` (Regan McGregor)

### Memory Files for Canonical IDs
Consult these memory files rather than hardcoding IDs:
- `<memory>/notion_user.md` — user UUID
- `<memory>/notion_teamspaces.md` — all 6 teamspace UUIDs
- `<memory>/notion_databases.md` — all DB IDs across teamspaces

### Notion Sync — `/publish` Skill
Notes synced to Notion carry YAML front matter:

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

- `canonical: obsidian` → Obsidian wins; push to Notion (engineering specs, project drafts)
- `canonical: notion` → Notion wins; pull into Obsidian for agent reads (brand voice, tone, competitor docs)

### Schema YAML — Canonical Tasks/Projects Schema
Canonical property schema for all Tasks + Projects DBs:

```
Departments/General/Context/tasks-schema.yaml
```

This file is `canonical: obsidian`. Published to the Tasks Schema Reference page in Notion (ID in `<memory>/notion_databases.md`).

### `/sync-tasks-schema` Skill
Propagates `tasks-schema.yaml` to all 12 dept Tasks + Projects DBs in Notion.

- `/sync-tasks-schema` — sync all 12 DBs
- `/sync-tasks-schema --dry-run` — preview without writing
- `/sync-tasks-schema --db <slug>` — target a single DB

**Limitations (manual Notion UI required):** status option names/colours and default Assignee cannot be set via the Notion API.

### Phase 2 Dept-Specific Databases
Seven new DBs added in Phase 2 (IDs in `<memory>/notion_databases.md`):

| DB | Teamspace |
|---|---|
| OKRs | General |
| Decisions Log | General |
| Content Calendar | Marketing |
| Social Posts | Marketing |
| Email/Newsletter | Marketing |
| Articles | Customer Service |
| Returns | Operations |

### Playbook Structure
Each dept has `Departments/<dept>/Playbooks/` in Obsidian, mirrored to a `📚 Playbooks` parent page in Notion. Parent page IDs in `<memory>/notion_databases.md`.

- Engineering playbooks: `canonical: obsidian`
- All other dept playbooks: `canonical: notion`

19 stub playbook pages exist across all 6 departments (skeletons — content TBD).

## Contact & Support

For questions about this codebase:
- Business tasks: Use Notion for task management (see teamspace IDs above)
- Technical issues: Refer to infrastructure documentation in `Departments/Engineering/Context/Infrastructure/`
- Code conventions: Follow patterns established in `n8n-utilities.js`</content>
<parameter name="filePath">/Users/regan.mcgregor/Library/Mobile Documents/iCloud~md~obsidian/Documents/TechLoop/AGENTS.md