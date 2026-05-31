---
notion_page_id: 3558c3d3-c03e-818f-aa2a-e91e479c93c4
notion_parent_id: 7093b771-b742-41cd-9642-49e0b3c6531c
notion_teamspace: engineering
canonical: obsidian
department: engineering
type: project-spec
synced_at: 2026-05-03T16:40:00Z
---
# TechLoop Inventory Automation

**Status:** ✅ Phases 1–6.2 Complete | 🚧 Phase 6.3 In Progress | 🚀 Production Active
**Linear Project:** [TechLoop Inventory Automation](https://linear.app/techloophq/project/techloop-inventory-automation-92cc3fc5a76f)
**Started:** 2026-01-12
**System Overview:** [[System Overview - TechLoop Inventory Automation|View System Documentation]]

---

## Overview

Decoupled inventory automation system syncing Leader supplier XML feed with WooCommerce using n8n and Supabase PostgreSQL.

**Philosophy:** "Raw over Modelled" - Ingest ALL data, name columns after source (snake_case)
**Architecture:** Decoupled workflows for Ingest → Mirror → Sync → Automation

---

## Current Status

**Phase:** Phase 6.3 In Progress 🚧
**Active Workflows:**

| # | Workflow | Schedule | Status |
|---|----------|----------|--------|
| 1 | Leader Ingest | Every 2h | ✅ Active |
| 2 | WC Mirroring | Every 4h | ✅ Active |
| 3 | Inventory Syncer | Every 3h | ✅ Active |
| 4 | Price Watchdog | Daily 6am | ✅ Active |
| 5 | Product Detector | Every 2h | ✅ Active |
| 6 | Product Publisher | Every 5m | ✅ Active |
| 7 | Slack Interaction Handler | Real-time | ✅ Active |
| 8 | Queue Reviewer | Every 4h + `/review-queue` | ✅ Active |

**Phase 6.3 Enrichment Workflows (7 of 9 live):**

| Workflow | Schedule | Status |
|----------|----------|--------|
| TL_URL_Discoverer | Daily 02:00 | ✅ Live |
| TL_Scraper | Every 2h | ✅ Live |
| TL_Mirror_WC_Attributes | Daily 02:00 | ✅ Live |
| TL_Enrich_Attributes | Every 15m | ✅ Live |
| TL_Attribute_Proposer | Daily 03:00 | ✅ Live |
| TL_Enrichment_Reviewer | Every 4h | ✅ Live |
| TL_Attribute_WC_Pusher | Every 10m | ✅ Live |
| TL_Enrich_Descriptions | Every 15m | ⏳ Planned |
| TL_Enrich_Images | Every 30m | ⏳ Planned |

**Recent Milestones:**
- 🚧 **2026-02-28:** Phase 6.3 (Enrichment) in progress — Attributes pipeline live, Descriptions and Images planned.
- ✅ **2026-02-15:** Phase 6.3 scraping and attribute extraction pipelines deployed.
- ✅ **2026-02-08:** Phase 6.2 (Product Automation) deployed. AI-optimised titles, descriptions, and short descriptions active via Ollama and Qdrant RAG.
- ✅ **2026-02-07:** Slack HITL (Human-in-the-loop) integration completed for approvals.
- ✅ **2026-01-25:** Added RRP tracking to Price Watchdog and clarified SQL schema.
- ✅ **2026-01-24:** Workflow 3 (Inventory Syncer) deployed and functional.
- ✅ **2026-01-24:** Phase 4 Price Watchdog built with Slack reporting active.
- ✅ **2026-01-13:** Phases 1–2 Foundation and Core Ingestion established.

---

## Core System Components

- **Supabase PostgreSQL:** Central data warehouse for supplier and store states.
- **n8n Automation:** Middleware handling API communication and logic.
- **Ollama (qwen2.5:14b):** Local AI for content enrichment.
- **Qdrant Vector DB:** RAG storage for title and description examples.
- **Slack:** HITL interface for product approvals and notifications.
- **Crawl4AI:** Playwright-based scraper for manufacturer pages.
- **Audit Logs:** `tl_sync_log` tracks all changes pushed to WooCommerce.

---

## Implementation Progress

### ✅ Phase 1: Foundation
- Supabase database schema (TEC-5)
- Leader XML feed validation (TEC-6)
- Database testing with sample data (TEC-7)

### ✅ Phase 2: Ingestion
- Workflow 1 — Leader Ingest (TEC-8)
- Workflow 2 — WooCommerce Mirroring (TEC-9)
- JavaScript utility library (TEC-10)

### ✅ Phase 3: Sync Logic
- Workflow 3 — Inventory Syncer (TEC-11)
- Error recovery (TEC-12)
- Discontinuation logic (TEC-13)

### ✅ Phase 4: Monitoring
- Workflow 4 — Price Watchdog (TEC-14)
- Slack alert system (TEC-16)

### ✅ Phase 6.1–6.2: Product Automation
- Workflow 5 — Product Detector
- Workflow 6 — Product Publisher
- Workflow 7 — Slack Interaction Handler
- Workflow 8 — Queue Reviewer
- Ollama AI title/description enrichment + Qdrant RAG
- Slack HITL approval system

### 🚧 Phase 6.3: Product Enrichment Suite
See [[Phase 6.3 - Product Enrichment Pipeline]] for full spec.

| Phase | Focus | Status |
|-------|-------|--------|
| Phase 0 | Scraping — Crawl4AI manufacturer data lake | ✅ Live |
| Phase 1 | Attributes — LLM extraction, HITL, WC push | ✅ Live |
| Phase 2 | Descriptions — feature-focused, LLM from markdown | ⏳ Planned |
| Phase 3 | Images — scraped images → WC gallery | ⏳ Planned |

### 🚧 Phase 5: Production & Maintenance
See [[Future Roadmap - TechLoop Inventory Automation]] for detail.
- Load testing (TEC-17)
- Operations runbooks (TEC-18)
- Data retention policies (TEC-19)

---

## Quick Links

- [Supabase Dashboard](https://supabase.reganmcgregor.com.au)
- [n8n Workflows](https://n8n.reganmcgregor.com.au)
- [[System Overview - TechLoop Inventory Automation|System Architecture & Workflow Catalog]]
- [[Phase 6.3 - Product Enrichment Pipeline|Phase 6.3 Full Specification]]
- [[Future Roadmap - TechLoop Inventory Automation|Roadmap & Maintenance Tasks]]
- [[3-Resources/Infrastructure/Database Schema - TechLoop Inventory|Core SQL Schema]]
- [[3-Resources/Infrastructure/Database Schema - Product Automation|Phase 6 SQL Schema]]

**Workflow Docs:**
- [[Workflows/Workflow 5 - Product Detector|Workflow 5 — Product Detector]]
- [[Workflows/Workflow 6 - Product Publisher|Workflow 6 — Product Publisher]]
- [[Workflows/Workflow 8 - Queue Reviewer|Workflow 8 — Queue Reviewer]]

**Config:**
- [[Config/Category Mapping Guide|Category Mapping Guide]]

---

## Project Archive

Historical planning, specifications, and implementation logs:

- **[[Archive/Workflows/|Workflow Implementations]]**: Original W1–W4 build logs and JSON exports.
- **[[Archive/Migrations/|Migrations]]**: One-time ACF migration documents.
- **[[Archive/Specs/|Specifications]]**: Original logic and architecture drafts.
- **[[Archive/Guides/|Legacy Guides]]**: Initial setup and quick-reference guides.
- **[[Archive/Logs/|Development Logs]]**: Code snippets and bug-fix records.
