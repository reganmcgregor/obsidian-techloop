# Scraper Replacement Proposal

**Date:** 2026-02-28
**Status:** Under Review
**Related:** [[Phase 6.3 - Product Enrichment Pipeline]] | [[System Overview - TechLoop Inventory Automation]]

---

## Background

The current `TL_Scraper` workflow uses a two-phase approach: a plain HTTP GET first (with a Chrome user-agent header), falling back to a Browserless Chromium Docker container if the response is empty or non-200. This setup is encountering four compounding issues:

- **High failure rate** — many products exhausting their 3 retries and landing in `scrape_status = 'failed'`
- **Anti-bot blocking** — sites serving CAPTCHAs, 403 responses, or bot challenge pages
- **JS content not loading** — Browserless returns the HTML shell but JavaScript-rendered content (spec tables, product details) is absent
- **Browserless instability** — the container crashes or times out under load

The Browserless container (`192.168.1.111:3100`) is a basic headless Chrome that provides rendered HTML but has no stealth mode, no proxy rotation, and is prone to memory issues.

---

## Options Compared

### Option A — Crawl4AI (Self-Hosted)

Crawl4AI is a purpose-built web scraping engine (50K+ GitHub stars, Apache-licensed). It uses Playwright under the hood, includes browser fingerprint spoofing, and exposes a REST API via Docker.

**How it integrates:**
Deploy Docker container on `192.168.1.111:11235`. n8n submits a crawl job (POST `/crawl`), waits ~10 seconds, then polls for the result. Returns both raw HTML and clean Markdown.

| | |
|---|---|
| **Self-hosting** | Production-ready Docker container, ~1 GB RAM, `--shm-size=1g` required |
| **n8n integration** | Standard HTTP Request node — no extra dependencies |
| **JS rendering** | Playwright Chromium with configurable wait conditions |
| **Anti-bot** | Built-in stealth mode (mimics real browser fingerprints, user-agent rotation) |
| **Output** | Raw HTML + clean Markdown (LLM-ready), images, screenshots |
| **License** | Apache 2.0 — no restrictions for commercial use |
| **Cost** | Free (self-hosted) |

**Pros:**
- Stealth mode dramatically reduces bot detection — the main cause of failures
- Playwright renders JS content fully before returning HTML
- Returns clean Markdown in addition to raw HTML — feeds LLM nodes (Enrich Attributes, Enrich Descriptions) directly without prompting them to parse HTML
- Stable, well-documented Docker deployment
- Removes the dual HTTP/Browserless path — simpler workflow (3 nodes replace 5)
- Built-in REST API dashboard for monitoring at `/dashboard`

**Cons:**
- Async API only — requires a Wait node + polling step in n8n (minor complexity)
- Heavier than Browserless (~1 GB RAM vs Browserless's lighter footprint)
- If a site uses sophisticated fingerprinting (e.g., Cloudflare Enterprise), stealth mode may still fail
- New dependency to maintain

---

### Option B — Firecrawl (Self-Hosted)

Firecrawl is a cloud-first scraping service that also publishes its source code. The self-hosted version uses Docker Compose.

| | |
|---|---|
| **Self-hosting** | Docker Compose available, but Firecrawl's own docs state it is *"not fully ready for production self-hosting"* |
| **n8n integration** | HTTP Request node or native n8n integration skill |
| **JS rendering** | Full support |
| **Anti-bot** | Proxy rotation infrastructure |
| **Output** | Markdown, HTML, JSON |
| **License** | **AGPL-3.0** — requires derivative works to be open-sourced |
| **Cost** | Free (self-hosted) or paid cloud (~$500+/month at scale) |

**Pros:**
- Clean Markdown output, well-documented API
- Native n8n integration skill available
- Paid cloud option exists as a fallback if self-hosting fails

**Cons:**
- **Self-hosting is explicitly flagged as not production-ready** by Firecrawl themselves
- **AGPL-3.0 licence** is problematic for commercial use — any integration could be considered a derivative work requiring open-sourcing
- Cloud pricing at TechLoop's scale (800+ products, ongoing re-scrapes) would be expensive
- Proxy rotation is cloud-side — not available in self-hosted version
- Less transparent on resource requirements for self-hosted

---

### Option C — Improve Existing Browserless Setup

Rather than replacing Browserless, tune the current setup to reduce failures.

**Potential improvements:**
- Increase timeouts and add explicit `waitUntil: networkidle` option
- Add a random delay between scrapes to avoid rate limits
- Add proxy support via a free/cheap residential proxy service
- Force all products through Browserless (skip plain HTTP fetch entirely)

**Pros:**
- No new infrastructure or dependencies
- Faster to implement — only workflow changes needed
- Lower memory overhead

**Cons:**
- Does not fix the root cause: Browserless has no stealth mode and is fingerprinted as a headless browser
- Proxy services add ongoing cost and another point of failure
- JS rendering issues will persist — Browserless fetches HTML at `DOMContentLoaded`, not after JS execution completes
- Instability is inherent to the single-container setup — no built-in recovery

---

## Summary Comparison

| Factor | Crawl4AI | Firecrawl (self-hosted) | Improve Browserless |
|---|---|---|---|
| Self-hosting readiness | Production-ready | Not production-ready | Already running |
| Anti-bot stealth | Built-in stealth mode | Proxy rotation (cloud only) | None |
| JS rendering quality | Playwright (excellent) | Full support | Basic (DOMContentLoaded only) |
| Output formats | HTML + Markdown | HTML + Markdown | HTML only |
| n8n integration effort | Medium (async polling) | Low (native skill) | Low (already built) |
| Licence | Apache 2.0 | AGPL-3.0 (⚠️ risky) | N/A |
| Cost | Free | Free / expensive cloud | Free |
| Maintenance burden | Low | Medium | Already known |
| LLM-ready output | Yes (Markdown) | Yes (Markdown) | No (raw HTML only) |

---

## Recommendation: Option A — Crawl4AI

Crawl4AI is the right choice for this use case. The primary failure causes (bot detection and incomplete JS rendering) are directly addressed by its stealth Playwright engine. The Apache 2.0 licence avoids the legal uncertainty of AGPL. Firecrawl's self-hosted version is explicitly flagged as unready by its own maintainers.

The additional benefit of clean Markdown output is significant: downstream LLM workflows (Enrich Attributes, Enrich Descriptions) currently receive raw HTML which forces the model to strip noise before extracting useful content. With Markdown stored in a new `markdown_content` column, those workflows can feed cleaner, smaller prompts to Ollama.

**Implementation scope:**
- Deploy Crawl4AI Docker container on `192.168.1.111:11235` (~15 min)
- Add `markdown_content TEXT` column to `tl_manufacturer_raw_data` in Supabase (~2 min)
- Update `TL_Scraper` workflow: remove 5 nodes, add 3, update Parse HTML and Prep Upsert (~30 min)
- No changes needed to downstream enrichment workflows (they still read `raw_html`; `markdown_content` is additive)
