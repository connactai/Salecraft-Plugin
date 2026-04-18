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

### #31 — `marketing-backend-staging` image has startup `NameError` on `config.py:712` 🔴 open (backend, env update blocked)
Any `gcloud run services update --update-env-vars` on `marketing-backend-staging`
spawns a new revision that fails STARTUP TCP probe with
`NameError: name 'settings' is not defined` at `config.py:712`
(`_seedream_settings = settings.get("seedream_image", {})` — should be
`_settings.get`). Cloud Run correctly keeps traffic on the last healthy
revision (`marketing-backend-staging-00403-tzt`) so the service is still
200-healthy from the outside, but any env change / secret rotation on
staging is frozen until a fresh image is built without the typo.

Impact: attempted to set `AI_TOKEN_ENABLED=true` on staging (to close
finding #3's staging-path 404) and could not deploy. Local
`marketing_backend/config.py` HEAD doesn't contain the bad line, so the
deployed image is from a partial / stale branch. Owner: marketing_backend
— rebuild + redeploy staging.

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
