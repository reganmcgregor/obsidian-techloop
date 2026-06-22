---
notion_page_id: null
notion_parent_id: null
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: null
---

# TechLoop Product Enrichment Engine

> Standalone Engineering project, spun out of [[TechLoop Shopify Migration]] (2026-06-22). Formerly tracked as "Phase 14 — Enrichment Service Rebuild". It is **enrichment** (improving product data/content), distinct from [[TechLoop Inventory Automation]] (operational actions like pricing/stock). **Pricing is not part of this project.**

## What it is

A continuous, incremental engine that enriches the **structured attributes/specs** of TechLoop's Shopify products — reading title + supplier feed + description (+ images, later), recommending Shopify-taxonomy-first attributes and values, and writing them to Shopify as metafields. Those metafields serve storefront **filters**, Shopify's **agentic catalogue**, and the **Google Merchant Center** feed.

Built to scale toward ~10,000 products across many categories (many empty today, filled over time). Products stay live regardless of enrichment state; the engine **gradually adds attributes and values** — never big-bang.

## Scope & roadmap

| Phase | Scope | Status |
|---|---|---|
| **P1 — Attributes & Specs** | Taxonomy categories + metafields (filters / agentic / GMC); collection assignment folds in with categories. | **in progress** |
| P2 — Descriptions | Rewrite product copy from extracted attributes + scraped manufacturer content. | deferred |
| P3 — Images | Curate/QA better images from scraped sources + vision. | deferred |
| (not here) Pricing | Repricing/margin actions = automation → [[TechLoop Inventory Automation]]. | out of scope |

## Locked decisions

- **Taxonomy-first, AI-proposed schema.** Per Shopify category, start from Standard Product Taxonomy attributes; mint `custom.*` only for genuine gaps. An LLM proposes the per-category schema; a human approves in Slack.
- **Model: single `gemma4:12b` (QAT) generalist** on the local Ollama (labgregor, RTX 3060 12 GB), thinking-off for extraction/tool-calling. Replaces `qwen2.5:14b`; multimodal so the (optional, deferred) image pass uses the same model. ✅ **Gate passed 2026-06-22** — see status.
- **Full Slack human review** before any metafield is pushed.
- **Scrape is not a prerequisite** for attributes (extract from title + feed + description; scrape re-triggers enrichment + feeds P2/P3).
- **Push is isolation-safe** (adaptive batched/per-metafield `metafieldsSet`) and **category-aware** (assign Shopify taxonomy category first; constrained-key mismatch routes to `custom.*`).

## Architecture (W0–W9)

Continuous overlay on the already-live catalogue:

`W1 sync def-mirror` + `W0 assign category (leaf-checked)` → `W2 propose schema → W3 Slack approve` → `W4 enable defs + seed vocab` → `W5 text-first extract (+W5b optional vision)` → `W6 Slack review → W7 approve→queued` → `W8 isolation-safe push` → `W9 health digest + Search & Discovery facets`.

Reuses the existing feed/scrape spine (`TL_Ingest_Leader_Feed`, `TL_URL_Discoverer`, `TL_Scraper`). Full design, data-model changes, build steps, pilot plan, and open decisions live in the execution plan:

**Execution plan:** `~/.claude/plans/departments-engineering-projects-phase-purring-quokka.md`

## Pilot categories

- **Desktop Memory** → `gid://shopify/TaxonomyCategory/el-7-12-3` (Memory > RAM) — text path + numeric attrs + custom gap (CAS).
- **Computer Cases** → `gid://shopify/TaxonomyCategory/el-7-9-8` (Desktop Computer & Server Cases) — vision path + TaxonomyValue/Metaobject resolution.

(Both confirmed `status='approved'` in `tl_shopify_category_map`. Leaf-node + attribute-existence verification folds into W1/W2.)

## Status — 2026-06-22

- ✅ Plan approved; project spun out of the migration.
- ✅ **Step 0 model gate passed.** `gemma4:12b` pulled on labgregor; validated head-to-head with `qwen2.5:14b` (identical correct extraction, clean non-thinking JSON) AND through the live n8n **AI Agent + Calculator tool + Structured Output Parser** path (`{result:42, used_calculator:true}`). Single-model Gemma adopted; fallback pair (qwen3:8b + gemma4:e4b) not needed. Reusable Ollama LangChain credential created: `bFORc9N56kqykD0i` ("Ollama (labgregor)").
- ⏳ **Blocked:** DDL migration + W1/W0 build/testing await the Supabase MCP connection (was intermittently unreachable from the build host on 2026-06-22). Taxonomy leaf check folds into W1/W2 when DB access is back.
- ⏭️ Next: stand up the Notion Projects-DB row (Engineering teamspace `3558c3d3-c03e-81f4-808c-00427b3b52e1`); apply additive Supabase DDL; build W1 then W0.

See [[Phase 14 — Enrichment Service Rebuild]] (legacy pointer), [[metafield_definition_drift]] memory for the known-issues backlog.
