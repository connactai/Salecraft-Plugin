# Dogfooding Findings — Plugin × Real LLMs

Living document. Each finding is what broke or surprised when a real LLM
(Gemini Web, Claude.ai with MCP, ChatGPT Plus, Claude Code) ran the
SaleCraft plugin against the production backend.

## Format

| # | Title | Severity | Status | Owner |
|---|-------|----------|--------|-------|
| Severity: 🔴 critical / 🟠 high / 🟡 medium / ⚪ low / informational
| Status: open / fixed / wont-fix / needs-clarification

---

## Findings

### #41 — `update_session` silently drops unknown top-level keys 🔴 critical / **open (backend fix recommended)**

**LLM behavior observed** (Claude.ai + MCP, dogfooding 2026-04):
LLM wrote brand text fields (`base_description` / `brand_description` / `value_proposition` / `brand_story` / `tagline` / `primary_color` / `key_features` / `cuisine_type` / `signature_dishes` / `operating_hours` / `pricing_info` / `target_audience` / `trust_certifications`) at the **top level** of `update_session.data` across 5 batches. Per-batch response was `isError: false` + `updated_at` bumped, so LLM reported "✅ written" each time. User spent ~20 minutes reviewing/editing the scraped content per batch (all 4 rounds of per-field ask-back).

After all 5 batches + user-approved TA selection + spokesperson generation + cost recital + user "開始", `generate_session` fired. Total charge: 4,200 pts across 2 TAs. LP generation completed. User opened frontend dashboard → saw 核心產品描述 / 產品規格 / 參考圖片 / 注意事項 all EMPTY. Backend session after inspection: only `product_name` + `wizard_shared_data.{images}` + `wizard_ta_groups[...]` had persisted. All 13 text fields the user approved were dropped at write time.

**Root cause**: `update_session` endpoint accepts only 6 whitelisted top-level keys (`session_name` / `product_name` / `wizard_shared_data` / `wizard_shared_files` / `wizard_ta_groups` / `wizard_ta_group_files`). Unknown top-level keys are silently dropped — no warning, no 400, no `extra_forbidden` error. The request succeeds (200 OK), `updated_at` advances (because the request itself is processed), but the content the LLM intended to write doesn't persist.

**Why LLM didn't catch it**: standard success signals (no exception, `success: true`, `updated_at` changed) are all TRUE even on silent drop. The only way to detect it is to `get_session` after every write and assert that the specific keys landed — which plugin docs did not previously require.

**Fix (plugin)**: new `update_session` whitelist documentation + MANDATORY per-key verify-after-write in CLAUDE.md rule 6.5, brand-onboard Phase 1, generate-landing batch-write verification. New 6.5 violation example with concrete failure narrative.

**Fix (backend, STRONGLY RECOMMENDED)**: add `extra="forbid"` to the `update_session` request schema. This converts silent drops into loud 422s, which LLMs cannot ignore. Matches the fix applied to ad-campaign schemas (#36 cascade) that eliminated an identical class of bug. Plugin-level "self-audit" defense is brittle — any new LLM or any distraction causes the same failure.

**User-reported (2026-04)**: "我 70% / 後端 30%。silently 丟棄未知欄位是典型反模式、應該 return warning 或 400 error 說『unknown field: base_description』。這會讓 AI 在第一次就發現錯誤、不會連錯 5 批。"

---

### #40 — `landing_ai_mcp` full docstring audit ⚪ clean (0 real drift)
Ran automated `scripts/audit_mcp_docstrings.py` against all 85
@mcp.tool() functions in `Service_system/landing_ai_mcp/tools/`
that accept a `*_json` parameter. Cross-referenced 992 known
Pydantic field names from `marketing_backend/routers/*.py` and
`marketing_backend/core/schemas.py`.

Result:
- **10 clean** (every named field resolves to a schema)
- **54 vague** (docstring says only "JSON with ... parameters" —
  doc quality backlog but not correctness)
- **21 drift-candidates** — manually verified all 21 are false
  positives:
  - enum values inline (awareness / conversion / meta)
  - example placeholder values (ta_1, url1, id1)
  - wrapper-tool-specific surface that maps to multiple backend
    endpoints (update_header.font_size → re-shaped into
    /stripes/0/text-styling call)
  - cross-references in "see also" sections (update_stripe_text ↔
    update_stripe_texts)

**No truly-wrong field names in `landing_ai_mcp` like the #30
generate_ad.platform drift.** The audit script
(`Service_system/scripts/audit_mcp_docstrings.py`, commit f699125)
is reusable — re-run on every schema change or new MCP tool.

### #39 — Meta orphan campaigns accumulated in test ad account ⚪ accepted / won't-clean
Pre-extra=forbid cascade left ~11 orphan campaigns in the test1
Meta Ads account (status=PAUSED, $0 spend). With #36 landed, no new
orphans accumulate. User decision 2026-04-19: do not run the
cleanup script — the orphans don't affect billing, just UI clutter
in Meta Ads Manager. `scripts/list_meta_orphans.py` is ready if
anyone changes their mind later.

### #38 — No `cancel_generation` endpoint 🟡 won't-fix (by design)
User decision 2026-04-19: generation cannot be cancelled mid-flight.
Once the pipeline starts, the cost has been committed (Gemini API
calls, Seedance GPU time, etc.). Adding a cancel path would promise
a refund we can't provide without losing money per call.

Acknowledged limitation — documented in Salecraft-Plugin SKILL docs
for LLMs to communicate clearly: "once generation starts, it runs
to completion; there is no cancel". Interrupted generations from
crashes / timeouts still get full refunds via
`core/refund_service.py`.

### #37 — `deducted_amount` ≠ final cost confusion ⚪ no-op (already correct)
Previous Claude reported `deducted_amount=2000` for a session with
`actual_stripe_count=8` and assumed the system was overcharging.
Root cause: `SessionTAGroup.deducted_amount` stores the
**pre-charge** (requested × cost_per_stripe), not the **net cost
after stripe_adjustment refund**. A snapshot taken before the
Factory pipeline finished will always show the pre-charge.

Verified via transaction log for test1@test.com:
- 18:43:11 pre-deduct: -2000 (10 stripes × 200)
- 18:50:27 stripe_adjustment: +200 (9 actual vs 10 requested)
- **net: 1800 = 9 × 200** ← Option A (「多生成無償、少生成補償」) working as designed.

No refund owed. Code in `core/credit_utils.py:calculate_stripe_credit_adjustment`
already implements the user's preferred policy:
- `actual >= requested` → no extra charge (free overdelivery)
- `actual < requested` → refund the difference

Finding downgraded from "pricing bug" to "confusing UI field name".
Future polish: expose a computed `net_cost` field on
`SessionTAGroupResponse` so clients don't have to do the math.

### #36 — `extra="forbid"` missing on publish/ads request schemas (root of #25/#26 cascade) 🔴 fixed (backend)
The chain that burned all three of #25/#26/#30 was:
1. MCP docstring listed wrong field names (`account_id`, `objective`,
   `creative_id`) — **fixed** in Service_system commits 1038be9 /
   5f9e834.
2. LLMs copied those names into `data_json`.
3. Pydantic schemas (`SocialPublishRequest` / `AdCampaignCreateRequest`
   / `AdsPromoteRequest`) did **not** set `extra="forbid"`, so the
   wrong keys were silently dropped, `campaign_objective` defaulted to
   `OUTCOME_TRAFFIC`, the AdSet `default_optimization_goal_for` picked
   `LINK_CLICKS`, and Meta rejected the resulting AWARENESS-campaign +
   LINK_CLICKS-adset combo.

**Fix (Zereo_backend commit dfde270)**: added
`model_config = ConfigDict(extra="forbid")` to `SocialPublishRequest`,
`SocialMultiPublishRequest`, `AdsPromoteRequest`, and
`AdCampaignCreateRequest`. All known callers (asia-agentic-commerence
TypeScript interfaces, zereo_social_mcp wrappers, Salecraft-Plugin
examples) already use correct field names, so this tightening is
safe. Verified: E2E test sending `account_id`/`objective`/`platform`
now returns 422 with specific `extra_forbidden` on the offending key
— no more silent drop.

Downgrades #25, #26, #30 from open to fixed by schema enforcement.

### #35 — `regenerate_stripe.user_feedback` CJK mojibake 🟡 diagnostic logging added
Dogfooding repeatedly reports `娘 → 岣` mojibake on the regenerate
path only. REST/MCP round-trips on other paths are clean. Root cause
is between DB-stored feedback text and Gemini prompt assembly, but we
need byte-level samples to isolate. marketing_backend commit 627658c
ancestor adds a diagnostic INFO log of the first 30 bytes (hex) of
user_feedback when it contains CJK. Next reproduction will show us
both what the HTTP body delivered and what we're storing. No
behavioral fix yet — see plugin `lib/api-reference.md` "Unicode in
data_json strings — narrow scope" for the current workaround.

### #34 — `update_stripe_text` (singular) vs `update_stripe_texts` (plural) naming trap 🟡 fixed (MCP docstring)
Two tools with nearly identical names take differently-shaped bodies
and different arg counts. LLMs confuse them, getting 422 errors on the
other's body shape. MCP docstrings for both tools now explicitly
cross-reference each other and describe the expected JSON shape
(object for singular, array for plural). See Service_system commit
27117ff.

### #33 — `publish_post` no image URL pre-flight 🟠 fixed (backend)
Meta rejected bad image URLs (non-image MIME, unreachable) with a
generic "Only photo or video can be posted" error, and users waited
~60s for Meta's error loop before seeing the problem. Zereo_backend
commit dfde270 adds a HEAD request pre-flight in
`routers/social_publish.py` that catches HTTP 4xx and non-image
`content-type` before handing off to Meta. Failures say exactly
which URL failed and why. Network hiccups in the pre-flight are
non-fatal (we let Meta have the final say). Verified: E2E test sends
`https://www.google.com/` as image_url and gets `status=failed,
error_message="Image URL content-type is 'text/html; charset=iso-8859-1',
not image/*. Meta rejects non-image URLs with a generic error."`.

### #32 — `can_publish=false` but FB page publish actually succeeds 🟠 fixed (backend)
The FB page capability check marked accounts `can_publish=False` when
Meta's `/{page_id}` response omitted the `tasks` field, even though
the same token could successfully publish. Plugin advice was to
ignore the flag (finding #18), but the flag was still misleading
mobile UI and LLM reasoning. Zereo_backend commit dfde270 extends the
fallback: if the response carries any identifying field
(`name`/`id`/`category`/`username`), trust OAuth-verified tokens and
mark `can_publish=True`. Dogfooding #18 downgraded from open to fixed.

### #31 — `marketing-backend-staging` image had startup `NameError` on `config.py:712` 🔴 fixed (backend)
Commit 3be018c ("feat: add Seedream 5.0 as primary image generation
provider with Gemini fallback") wrote `settings.get("seedream_image", {})`
at `config.py:712`, but the module-level dict is named `_settings`
(underscore prefix). Every Cloud Run startup crashed in config.py before
FastAPI could bind PORT=8080, so any `gcloud run services update` on
`marketing-backend-staging` was rejected at the STARTUP TCP probe.

Cloud Run correctly kept traffic on the last healthy revision, so the
service stayed 200-healthy externally, but this blocked every env
change for hours — including setting AI_TOKEN_ENABLED=true on staging
to close finding #3's staging-path 404.

**Fix (commit 627658c)**: replace `settings.get` with `_settings.get`
on line 712, matching the pattern on line 289. Verified: subsequent
`services update --update-env-vars=AI_TOKEN_ENABLED=true` succeeded,
staging revision `00408-vjh` live.

### #30 — `generate_ad` schema rejects `platform` field 🟠 fixed (plugin doc + MCP docstring)
`/sessions/{id}/generate-ad` (`GenerateAdRequest`) has
`model_config = ConfigDict(extra="forbid")` and its valid fields are
`ta_group_id` (required), `aspect_ratio` (9:16/4:5/1:1, default 9:16),
`ad_goal` (awareness/traffic/conversion, default awareness). Three
places in the plugin — `skills/publish-ads/SKILL.md` Phase 3,
`skills/publish-social/SKILL.md` "快速生成廣告圖", and
`lib/mcp-patterns.md` — told LLMs to pass `{"platform": "meta",
"ta_group_id": "ta_1"}`. Every LLM following those examples would
receive `HTTP 422 extra_forbidden on "platform"`.

**Fix**: updated all three plugin locations + `lib/api-reference.md` +
the MCP docstring in `Service_system/landing_ai_mcp/tools/sessions.py`
so the tool definition LLMs discover at runtime also lists the correct
fields. Platform choice happens later (`create_ad_campaign`,
`publish_post`, `promote_reel`), not here.

### #1 — Sprint Plan guard not strong enough 🟠 fixed (plugin)
LLMs default to "be helpful, write the strategy" when user asks for paid
execution, instead of triggering the AI Token flow first. **Fix**: added
the `🚨 FIRST-RESPONSE RULE` at the top of `CLAUDE.md` — when user uses a
"do" verb on a paid output, the AI's first reply must be ONLY (1) cost
sentence (2) AI Token 3-step prompt (3) optional ≤1-line scope question.
No Hero Section text allowed in this turn.

### #2 — Brand authorization check missing 🟡 needs-clarification
Originally raised by Claude.ai but without specifics. Possible meaning:
the AI should verify the user has the right to use a brand's name/assets
before generating LPs (e.g., shouldn't generate "Apple iPhone LP" for
random user). Needs follow-up dogfooding to confirm.

### #3 — `authenticate_with_token` MCP tool returns 404 instead of 401 🔴 fixed (backend env)
Root cause: `marketing-backend-staging` was missing the
`AI_TOKEN_ENABLED=true` env var, so `/auth/ai-token/exchange` returned
404 instead of 401 INVALID_AI_TOKEN. The `service-system-staging` MCP
proxy faithfully relayed the 404, so MCP-using LLMs saw `NOT_FOUND`
errors and got confused (one diagnosed it as "JWT secret mismatch" —
wrong direction). **Fix**: added env var to staging Cloud Run service +
forced traffic to latest revision. Verified B站 now returns proper 401.

### #4 — Staging vs production token isolation has misleading errors 🟡 wont-fix
Reported by Claude.ai but turned out to be wrong diagnosis: A站 and B站
share the same Cloud SQL DB and same JWT `SECRET_KEY` (both via
GCP Secret Manager). Tokens issued from either side validate on either
side identically. The only difference users see is the URL hash format
in Cloud Run URLs (`s6ykq3ylca-de` vs `876464738390` — both route to
the same service). No real environment-isolation issue exists.

### #5 — `list_pricing_plans` requires `user_token` 🟠 open (backend)
Pricing should be public information so AI agents can quote costs during
the pre-paid consultation phase. Currently AI must wait until the user
authenticates to even tell them what things cost. Backend fix: make
`/pricing/plans` (and similar read-only pricing endpoints) public.

### #6 — No lightweight `/auth/validate` endpoint 🟠 open (backend)
After exchanging a `sc_live_*` for an `access_token`, the AI has no
cheap way to confirm the token is still valid + see remaining scope/credits
without calling `/auth/me` (which loads full user profile). For polling
and pre-flight checks, a `/auth/validate` returning
`{valid: bool, scope: str, credits: int, expires_at: str}` would let the
AI verify before committing to a paid call.

### #7 — Rung 2/3 sandbox egress allowlist not handled 🟠 fixed (plugin)
Claude.ai's bash and Python sandboxes have a domain allowlist that
excludes `*.run.app`, blocking direct REST to the SaleCraft backend
even though the AI has the right tool. The original capability ladder
assumed all bash/python environments allow arbitrary HTTPS, which isn't
true in commercial chat products. **Fix**: added Rung 2.5 to `CLAUDE.md`
— LLMs first probe whether the sandbox can reach `*.run.app`; if not,
they tell the user explicitly which platforms work (Claude Code,
Cursor, Cline, Gemini Code Execution) instead of silently failing or
generating curl for the user to run.

### #8 — (folded into #3)

### #9 — `brand_name` stays `None` when `create_session` lacks `brand_id` 🟡 fixed (plugin doc)
`POST /sessions/` doesn't auto-create a brand from `brand_name`/
`product_name` payload — it only joins to an existing brand via
`brand_id`. If LLM skips `POST /brands/` and goes straight to
`create_session`, the session has `brand_name=None`, downstream agents
lose brand context, and resulting LP visuals come out generic.
**Fix**: added "Pre-flight gotchas" section to `skills/generate-landing/
SKILL.md` documenting the required order: `analyze_brand_url` → `POST
/brands/` → `create_session(brand_id=...)` → ...

### #10 — LP time estimate hallucinated as "1-3 minutes" 🟡 fixed (plugin)
Claude.ai told the user "Pipeline 會跑約 1-3 分鐘" when the plugin
actually documents `~30 min` for full LP generation in 4 places
(CLAUDE.md L515, L567, L577, L691). Likely confused with Quick Ad
(~5 min) or with the Strategist+Architect early phases (which are
fast). **Fix**: time guidance was already in plugin; no further
plugin change needed. This is an LLM hallucination problem the plugin
can't fully prevent — but the FIRST-RESPONSE rule's cost sentence
should now also include time estimate to anchor expectation.

### #11 — Carousel `num_images` parameter typo silently uses default 🔴 fixed (plugin doc + pending backend)
LLM (Claude.ai) sent `{"stripe_count": 10, ...}` to `generate_carousel`
expecting 10 slides. Backend's `GenerateCarouselRequest` Pydantic model
doesn't set `extra="forbid"`, so the unknown field was silently dropped
and `num_images` defaulted to 5. User got a 5-slide carousel for
800 pts instead of the 10-slide they expected (1,300 pts) — and the
LLM didn't notice until after the carousel completed.
**Fix (plugin)**: documented exact field name in `lib/rest-api-direct.md`
and `skills/generate-landing/SKILL.md` pre-flight gotchas.
**Fix (backend, pending)**: add `extra="forbid"` to all paid request
models so wrong field names return 422 immediately.

### #29 — `creative_id` not resolved by `create_ad_campaign`; caller forced to pass `creative_image_url` 🟡 open (backend)
Plugin doc + schema suggest passing `creative_id` (returned by
`upload_ad_image` / `upload_ad_video`) to `create_ad_campaign`. In
practice the route handler doesn't look up the creative by id; it
expects raw `creative_image_url` / `creative_video_url`. AI agents
get a confusing "missing creative" error after dutifully calling the
upload step.

**Fix recommendation**: either (a) add creative_id resolution path
in social_ads.py route handler, or (b) update plugin doc + Pydantic
schema to remove the misleading creative_id reference.

### #28 — Plugin doc says `account_id`, backend wants `social_account_id` 🟡 open (plugin doc)
`skills/publish-ads/SKILL.md` line 179 references parameter
`account_id`; the actual Pydantic schema field is `social_account_id`
(matches publish_post for consistency). AI agents trying to follow
the doc get 422 validation errors on the first try.

**Fix**: rename the doc reference to match the schema. Same gotcha
as #11 (carousel num_images) — plugin doc and backend schema must
stay in lockstep, ideally auto-generated from one source of truth.

### #27 — Failed campaigns leave orphans in Meta ad account; no cleanup endpoint 🟠 open (backend)
When `create_ad_campaign` succeeds at the Campaign step but fails
at the AdSet step (per #25), Meta has already created an empty
Campaign shell. The backend records `status=failed` in its DB but
doesn't call DELETE on the orphan Meta campaign — and no
`cleanup_failed_campaigns` endpoint exists.

In dogfooding 2026-04-18, ~10 orphan campaigns had piled up in
test1's ad account (`act_642463704665856`) since 2026-03-25.
They don't spend money (no AdSet = no impressions) but pollute
the user's Meta Ads Manager UI.

**Fix recommendation**: in the route handler's exception path,
when AdSet creation fails after Campaign succeeded, call
`account.api_call('DELETE', f'/{platform_campaign_id}')` to roll
back. Add a `POST /social/ads/cleanup-orphans` endpoint for
historical cleanup.

### #26 — `campaign_objective` appears overwritten in DB record 🟡 needs-investigation
Claude.ai reported that passing `OUTCOME_AWARENESS` resulted in DB
record showing `OUTCOME_TRAFFIC`. Code review of route handler
doesn't show an obvious overwrite — `body.campaign_objective` is
passed directly to both `create_campaign(objective=...)` and the
DB insert. Possible explanations:
1. The DB record was auto-default-populated before user request
   bound (possible but unlikely)
2. `AdCampaignCreateRequest.campaign_objective` Pydantic default
   `"OUTCOME_TRAFFIC"` was applied because the LLM payload was
   silently rejected by an earlier validation layer
3. Different code path inserted the DB row

Needs more dogfooding evidence (full request/response capture) to
confirm whether this is a real bug or Claude.ai's misread.

### #25 — `create_ad_campaign` AdSet step always defaults `optimization_goal` to LINK_CLICKS 🔴 P0 fixed (backend bcfbaf1)
`routers/social_ads.py:360` called `create_adset()` without passing
`optimization_goal`, so the helper's default `"LINK_CLICKS"` was
always used. Meta API rejects LINK_CLICKS on
`OUTCOME_AWARENESS` / `OUTCOME_LEADS` / `OUTCOME_SALES`
objectives — those need REACH, LEAD_GENERATION, OFFSITE_CONVERSIONS
respectively.

Result: every non-TRAFFIC campaign failed at AdSet creation, leaving
an empty Campaign shell in the user's Meta ad account. Confirmed
~10 orphan campaigns piled up in the test ad account from
2026-03-25 to 2026-04-18 — bug had been live ~25 days.

**Fix (backend bcfbaf1)**: added `default_optimization_goal_for()`
helper in `meta_ads_publisher.py` with the per-objective safe-default
mapping. Route handler now derives the value from
`body.campaign_objective` and passes it explicitly to `create_adset`.

After deploy: AWARENESS / LEADS / SALES campaigns should reach the
AdSet step successfully. (Existing orphans are Meta-side state and
must be deleted manually via Meta Ads Manager — see #27 for the
cleanup-endpoint follow-up.)
After successful publish to IG/FB via `publish_post`, the only available
post-management endpoint is `cancel_post`, which only cancels posts
with `status=scheduled` (not yet sent to Meta). Once Meta has accepted
the post and stored a `platform_post_id`, the backend has no way to
delete it.

This becomes critical when an AI agent publishes erroneous content
(wrong brand, hallucinated stats, wrong account). The user has to open
the Instagram or Facebook app and manually delete each post — there's
no programmatic rollback.

Confirmed during dogfooding 2026-04-18 when an AI agent published 4
posts (2 IG + 2 FB) with hallucinated metrics; no API existed to undo.

**Recommended fix**: add `DELETE /social/publish/{post_id}` that, when
post status is `published`, calls Meta Graph API DELETE on the
platform_post_id. Should require explicit `confirm=true` query param to
avoid accidental deletes. Log all deletes prominently.

### #19 — No brand-ownership / authorization guardrail in the entire publish pipeline 🔴 P0 legal
The plugin pipeline `analyze_brand_url` → `create_brand` →
`create_session` → `generate_session` → `publish_post` has zero steps
that verify the user owns or is authorized to use the brand they're
generating content for. An AI agent can:
1. Scrape a competitor's website with `analyze_brand_url`
2. Build a brand profile from it
3. Generate full Landing Pages, ads, and social posts
4. Publish them to the user's own social accounts under that competitor's brand name

In dogfooding 2026-04-18 this was demonstrated end-to-end with the
微熱山丘 brand without any consent — 4 posts went live publicly.

Legal exposure: trademark infringement (商標法), unfair competition
(公平交易法), false advertising (公平交易法第 21 條, especially when
the LP contains fabricated metrics — see finding #9).

**Recommended fix (multi-layer)**:
- `create_brand`: require user to attest brand ownership (boolean +
  text reason). Log + audit.
- `analyze_brand_url`: when source URL domain doesn't match user's
  registered domain, mark the resulting brand as `ownership_unverified`
  and tag downstream sessions accordingly.
- `publish_post`: refuse to publish if `brand.ownership_verified=False`
  unless user explicitly bypasses with `i_attest_authorized=true` query
  param. Even then, log loudly.
- Top of plugin `CLAUDE.md`: hard rule that AI must ask user to confirm
  brand authorization before any `publish_*` call when the brand was
  auto-created from analyze_brand_url within the same session.

### #18 — `get_account_capability.can_publish` flag misaligned with actual publish capability 🟠 fixed (backend logic, pending)
FB Page reports `can_publish: false` and `tasks: []` (no MANAGE /
CREATE_CONTENT permissions visible to the API), yet `publish_post`
calls Meta Graph API successfully and the post goes live.

Likely cause: capability check queries the page's `tasks` field via the
user-level access token, but the publish path uses the page's own
long-lived page access token which has different (broader) scopes.

This misleads AI agents into refusing to publish when they actually
could — and worse, into publishing when they thought they couldn't,
which becomes the user's surprise.

Confirmed dogfooding 2026-04-18: `can_publish: false` on FB Page
"測試" but 2 posts still published successfully.

**Recommended fix**: align capability check with actual publish path.
Either (a) query capability with the page token (matches publish path),
or (b) attempt a dry-run publish for capability detection.

### #14 — `zereo_social_posts.image_urls` column missing → all publish endpoints 500 🔴 fixed (DB migration)
Triggered when AI agent called `publish_post` / `publish_multi` /
`publish_content` with a properly-formed payload. Backend logs showed:
`sqlalchemy.exc.ProgrammingError: column zereo_social_posts.image_urls
does not exist. Hint: Perhaps you meant to reference the column
"zereo_social_posts.image_url".`

Root cause: `Zereo_backend/migrations/add_carousel_image_urls.py` exists
in repo (`ALTER TABLE ... ADD COLUMN IF NOT EXISTS image_urls JSONB
DEFAULT '[]'::jsonb`) but was never run against the shared Cloud SQL
database. The Pydantic schema, ORM model, and route handler all
reference `image_urls`, but on every `INSERT` PostgreSQL rejected
the column → 500 Internal Server Error → `publish_post` /
`publish_multi` / `publish_content` all blocked.

Severity rating P0 because:
- All carousel publishes blocked across A站 + B站 (shared DB)
- 500 errors leaked stack traces (would also have blocked single-image
  posts that send `image_urls: []` from frontend)
- No way for AI agent to recover via retry (no schema change can fix
  it from client side)

**Fix**: ran the migration directly against the shared Cloud SQL DB
via `cloud-sql-proxy` + `pg8000`. Idempotent + additive. Pre-check
showed column missing; post-check confirmed `jsonb DEFAULT '[]'`.
Verified `publish_post` now returns 404 "Social account not found"
(proper error for invalid id) instead of 500. Both A站 and B站
benefit immediately since they share the DB.

**Open follow-up (backend team)**: investigate why `add_carousel_
image_urls.py` migration was never run on production. Options: add
to startup migration runner, add to CI/CD pipeline, document
explicit migration runbook.

### #13 — Public LP URL: wrong domain AND wrong path 🔴 fixed (plugin)
Plugin docs showed the public LP URL as
`https://salecraft.ai/{locale}/landing-page?id=...`. **Two bugs**:
1. Wrong domain — `salecraft.ai` is the brand/marketing site (where the
   marketingx token page lives), NOT the LP renderer. LPs are served by
   `marketing-frontend` Cloud Run service mapped to `landingai.info`.
   `salecraft.ai/.../landing-page?id=...` returns 404.
2. Wrong path — even on the correct domain, `/landing-page?id=X` is the
   legacy form that 301-redirects. The canonical user-facing URL is
   `/lp/<X>` (path param, no query string).

Backend already returns the legacy form internally
(`routers/landing.py` line 7958: `f"{base_url}/landing-page?id={...}"`),
which works via redirect — so end users don't break, but URLs we hand
out look uglier than necessary.
**Fix (plugin)**: every `salecraft.ai/{locale}/landing-page?id=...` →
`landingai.info/{locale}/lp/<id>` across CLAUDE.md, generate-landing,
edit-landing, homepage-builder skills. URL discipline rule (Core Rule
\#17) updated to allow both domains for distinct purposes.
**Fix (backend, optional)**: change `published_url` to canonical
`f"{base_url}/{locale}/lp/{project.id}"` so legacy redirect can be
deprecated later.

### #12 — `generate_session` defaults to 10 stripes when `stripe_count` omitted 🟠 fixed (plugin doc)
Same class of bug as #11: LLM created a session intending 8-page LP
(1,600 pts/TA quote it gave the user) but didn't pass `stripe_count`
to `generate_session`. Backend defaulted to 10 stripes (2,000 pts/TA),
overcharging by 400 pts/TA × 2 TAs = 800 pts in this run.
**Fix (plugin)**: pre-flight gotcha documented; FIRST-RESPONSE rule
now forces the AI to ask "8-page or 10-page?" before triggering and
to pass `stripe_count` explicitly.
**Fix (backend, optional)**: same `extra="forbid"` hardening +
consider making `stripe_count` required (no default) so callers
can't accidentally over-spend.

---

## How to add a new finding

When you (an LLM or human) hit a new edge case during dogfooding:

1. Append a new entry to this file with the next # in sequence.
2. Severity + status badges at the top.
3. Brief root-cause explanation.
4. If you fixed it: note where (plugin / backend / both) and what you
   changed. If pending: note who owns the fix and approximate effort.
5. Don't include AI Tokens, real test account emails, real user data,
   or stack traces with internal hostnames in this file.
