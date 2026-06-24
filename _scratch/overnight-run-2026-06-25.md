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
- **STEP 1 seed BUILT + RUNNING.** Generalised `aWG6RGF70QUfETWG` → `TL_Seed_Standard_Vocab_StoreWide` (queue-driven, atomic claim FOR UPDATE SKIP LOCKED, per-row error isolation, records GID→`tl_shopify_metaobject_values` + flips `tl_category_schema.vocab_status`). Seed queue `tl_vocab_seed_queue` = 3847 rows / 269 keys from taxonomy repo. Smoke-tested 5 ✓ (real GIDs). Full drain firing; at ~92/min, 0 errors. Webhook `seed-empty-standard-vocab`.
- **STEP 2 DONE.** Custom registry verified: all 17 keys are scalar (measurement/int/bool/text), ZERO metaobjects — owner directive already satisfied. Artefact keys (capacity-gb/latency/fan-count-front|rear/error-correction-*) are extraction-only, NOT registry defs → push gate excludes them.
- **STEP 3 PREPPED.** Extractor `9wOcrhnxwS3HaSHm`: store-wide-ready (generic batch, accuracy-first prompt uses seeded vocab). FIXED a latent bug: Upsert `ON CONFLICT` was 3-col but live unique index is 4-col `(shopify_product_id,attr_key,schema_version,normalized_value)` → would error on multi-select; now 4-col. Configured batch=10, schedule every 2 min. KEPT INACTIVE until seed done. Before activating: DELETE stale un-approved rows (742 in_review + 1 pending, schema_version=2, push_status='none') so they re-extract clean.
- **STEP 4 DESIGNED.** W8 push `9Av2xKgxSt7PXBRt` (active, webhook `tl-attribute-pusher-v2`, also self-pushes every 5min). Push gate = UPDATE→approved+queued WHERE schema_version=2 AND push_status='none' AND attr_key NOT LIKE '\_%' AND ((shopify AND value matches a seeded metaobject display_value) OR (custom AND attr_key IN registry)). W8 resolves standard→GID (aggregates multi-select into one metafield), formats measurements {value,unit}, retries per-metafield (constraint mismatch fails in isolation).
- WAITING on seed poller (bg task b8pki13gz) to finish → then DELETE stale → activate extractor → drain ~900 products across 74 cats → push gate → fire W8 → verify → morning report.
- **STEP 1 SEED ✅ DONE.** 3828 seeded / 19 errors (all color-pattern — special metaobject schema has no `taxonomy_reference` field; already has 19 vals from pilot, benign). Vocab now = **296 keys / 4333 values** (was 88 keys); 1253 schema rows flipped `seeded`.
- Cleared **839 stale rows** (pre-tuning in_review/pending/skipped_no_schema). Remaining v2 = 279 pushed/approved (live) + 4 skipped/failed.
- **STEP 3 EXTRACTION ✅ RUNNING (after fixing a 2nd latent bug).** First scheduled run errored: `Route on Parse Status` Switch had `fallbackOutput: 1` but only 1 rule → "output 1 not allowed" → whole batch lost whenever any item needs the retry path (RAM never hit it = clean JSON). Fixed → `fallbackOutput: 'extra'`. Re-verified: 123 rows / 10 products / 9 cats per cycle, multi-select one-row-per-value, vocab-constrained, quality excellent (motherboards/keyboards). Draining ~10 products/2min; ~920 remaining (~3 hr). Monitor: bg poller.
- NEXT (auto, when drain poller exits): push gate (queue exact-vocab + registry-custom rows) → fire W8 `tl-attribute-pusher-v2` → verify cross-category live spread → morning report.
- **STEP 4 PUSH ✅ VALIDATED EARLY (live).** Ran the gate on extracted-so-far: queued 266 rows / 18 products / 16 cats (98% vocab-match; only 5 no-match held). Fired W8 → **193 metafields LIVE across 13+ categories** (Laptops 71, Motherboards, Video Cards, Desktops, CPU Coolers, HDDs, Monitor Arms, NICs, Keyboards, CPUs, Hubs&Switches, Cable Trays), **44 failed in isolation = all `connection-type`** ("Owner subtype does not match" category-constraint) + 1 `monitor-screen-specialized-features`. Isolation works perfectly. The 29 'none' stragglers = benign race with the still-running extractor (re-touched→pending), ALL join to mirror fine (no silent drop). FULL CHAIN PROVEN on new categories.
- **FINAL-PUSH PLAN (when drain done):** (1) DEACTIVATE extractor first (avoid the gate/extract race). (2) Final gate = same confident filter BUT exclude (taxonomy_gid,attr_key) combos with a prior `failed` Owner-subtype row (auto-skip connection-type etc.). (3) Fire W8 / let its 5-min schedule push. (4) Verify spread, write report. connection-type/monitor-screen-specialized-features = W8 custom-routing follow-up (not tonight).
- LEFT RUNNING: extractor schedule (draining), drain poller bg `bqgldykq8`, W8 active (schedule 5min). Seeder left active (idle, queue empty).
