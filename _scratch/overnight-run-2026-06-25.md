# Overnight Enrichment Run — 2026-06-25 → 26

> Autonomous long-running agent session. Goal (owner, before bed): **seed the whole catalogue's standard vocab** + **enrich a broad cross-category sample of products live** so the owner wakes to real results spanning many categories. Quality gate on live push: only exact-vocab-match standard values + confident custom measurements; ambiguous → queued for review.

## Owner design directives (this session)
- Icecat = **secondary** (low Open-tier coverage); gemma4 from supplier descriptions is the primary engine tonight.
- Custom **metrics stay scalar** (measurement/int/bool), never metaobject lists. Verify + prune legacy/artefact keys.
- **Do NOT** build an auto-add-vocab workflow tonight. Missing values → `Other`/queued; owner adds by hand later.

## Pre-flight (✅ all green, 2026-06-25 night)
- SSH `reganmcgregor@labgregor` + psql ✅ · gemma4:12b-it-qat + GPU (3GB free) ✅ · disk 253GB ✅
- taxonomy repo on Mac ✅ · n8n API healthy ✅ · `TL_Attribute_Pusher_v2` active, Slack handler active ✅
- Universe: **984 products categorised across 74 categories**, all with supplier descriptions. 33 cats have some extraction, 41 zero.

## Plan / progress
- [ ] **STEP 1 — Store-wide vocab seed.** 269 pending standard metaobject keys / 108 cats. Prep `tl_vocab_seed_queue` from repo → generalise proven seeder (`aWG6RGF70QUfETWG`) to drain → verify.
- [ ] **STEP 2 — Custom registry audit/fix.** Confirm metrics are scalar; prune `capacity-gb`/`latency`/`fan-count-front|rear`/`error-correction-*` artefacts.
- [ ] **STEP 3 — Cross-category extraction.** gemma4 across the 74 categories (accuracy-first prompt).
- [ ] **STEP 4 — Confident cross-category push (live).** Queue exact-match + confident custom rows → fire `9Av2xKgxSt7PXBRt`.
- [ ] **STEP 5 — Morning report** (this file + phase14_status.md memory).

## Key IDs
- Seeder to generalise: `aWG6RGF70QUfETWG` (TL_Seed_Empty_Standard_Vocab). Extractor: `9wOcrhnxwS3HaSHm` (TL_Extract_Attributes, inactive). Push: `9Av2xKgxSt7PXBRt` (active, webhook `tl-attribute-pusher-v2`). Reviewer W6: `uLVSPdnRw7McVLH2`. Creds: Shopify `4Kf8ZBknzZl6bvvi`, Postgres `BSoGuZ9BOv4OWqXf`, Ollama `bFORc9N56kqykD0i`.
- Shopify GraphQL: `https://techloop-7.myshopify.com/admin/api/2025-07/graphql.json`; token via Get Token (client_credentials, cred 4Kf8ZBknzZl6bvvi).

## Log
- (start) Pre-flight complete; building STEP 1.
