# SaleCraft API Reference — MCP Tool Parameters

> **Source of truth for parameter names and required fields.** This doc is
> cross-checked against the real Pydantic schemas every time it's updated.
> When in doubt, trust this page over examples elsewhere.

LLMs reading a SKILL.md example that conflicts with this file should
**follow this file** — SKILL.md examples are illustrative, this reference
is the contract.

---

## Golden rules (read before calling any tool)

1. **`social_account_id` is the body field**, **`account_id` is the tool
   argument** — they look interchangeable, they are NOT:
   - Tool args (path params): `refresh_account_token(user_token, account_id)`,
     `disconnect_account(user_token, account_id)`, `recheck_capability(user_token, account_id)`
     → the MCP wrapper turns `account_id` into a URL path segment.
   - Body payloads (inside `data_json` or `targets_json`): **always use
     `social_account_id`** for the publish / ads endpoints. Passing
     `account_id` in a body gets you `422 field required: social_account_id`.

2. **`creative_image_url` (a URL string), not `creative_id` (an opaque id)**
   — `generate_ad` / `get_ad_result` returns `image_url` / `video_url`,
   and that URL is what `create_ad_campaign` consumes.

3. **Enum values take the full form with prefix**:
   - `campaign_objective`: `OUTCOME_TRAFFIC` (not `TRAFFIC`, not `CONVERSIONS`)
   - `post_type`: `ig_post / ig_story / ig_reel / fb_post / fb_story / tt_video / tt_photo`

4. **`target_genders` is list of ints**: `[0]=all / [1]=male / [2]=female`
   (not strings like `"male"`).

5. **`daily_budget` is USD float**, minimum `1.0`. Platform converts
   internally; don't pass `"currency"`.

---

## Publishing — `zereo_social_mcp`

### `publish_post` — single platform

MCP: `publish_post(user_token, data_json)` → the `data_json` string must
parse into `SocialPublishRequest`:

| Field | Required | Type | Default | Notes |
|-------|:--------:|------|---------|-------|
| `social_account_id` | ✓ | string | — | from `list_accounts` |
| `post_type` | ✓ | string | — | `ig_post / ig_story / ig_reel / fb_post / fb_story / tt_video / tt_photo` |
| `caption` | — | string | `null` | |
| `hashtags` | — | list[string] | `null` | |
| `image_url` | for image types | string | `null` | single image |
| `image_urls` | IG carousel (2–10) | list[string] | `null` | auto-syncs to `image_url[0]` if omitted |
| `video_url` | for video types | string | `null` | IG Reels, TikTok videos |
| `cover_url` | — | string | `null` | for IG Reels |
| `share_to_feed` | — | bool | `true` | IG Reels only |
| `privacy_level` | TikTok only | string | `null` | `PUBLIC_TO_EVERYONE / MUTUAL_FOLLOW_FRIENDS / FOLLOWER_OF_CREATOR / SELF_ONLY` — **sandbox force-downgrades to SELF_ONLY** |
| `disable_duet` | TikTok only | bool | `null` | |
| `disable_stitch` | TikTok only | bool | `null` | |
| `disable_comment` | TikTok only | bool | `null` | |
| `source_type` | — | string | `null` | `quick_ad / reels / manual` (tracking) |
| `source_session_id` | — | string | `null` | tracking |
| `scheduled_at` | — | ISO datetime | `null` | `null` = immediate publish |

Response (`SocialPublishResponse`):
`{ id, post_type, status ("draft"/"publishing"/"published"/"failed"/"skipped"/"cancelled"), platform_post_id, platform_permalink, error_message, scheduled_at, published_at, created_at, has_qr_pending }`

### `publish_multi` — fan-out

MCP: `publish_multi(user_token, targets_json)` where `targets_json`
parses into `List[SocialPublishRequest]` (same field rules as above
applied to each item).

### Platform-specific behavior quick table

| post_type | image_url | image_urls | video_url | Aspect | Notes |
|-----------|:---------:|:----------:|:---------:|--------|-------|
| `ig_post` | single | 2–10 carousel | — | 1:1 or 4:5 | carousel uses `image_urls` |
| `ig_story` | either | — | either | 9:16 | |
| `ig_reel` | — | — | ✓ | 9:16 | optional `cover_url`, `share_to_feed` |
| `fb_post` | either | — | either | any | |
| `fb_story` | either | — | either | 9:16 | |
| `tt_video` | — | — | ✓ | 9:16 | see sandbox rules in `skills/publish-social/SKILL.md` |
| `tt_photo` | ✓ | 2–35 | — | — | **JPG/JPEG/WEBP only, no PNG**; `PULL_FROM_URL` only (verified domain) |

---

## Ad Campaigns — `zereo_social_mcp`

### `create_ad_campaign`

MCP: `create_ad_campaign(user_token, data_json)` → `AdCampaignCreateRequest`:

| Field | Required | Type | Default | Notes |
|-------|:--------:|------|---------|-------|
| `social_account_id` | ✓ | string | — | ⚠️ NOT `account_id` |
| `campaign_name` | — | string | `""` | empty = auto-generated |
| `campaign_objective` | — | string | `"OUTCOME_TRAFFIC"` | `OUTCOME_AWARENESS / OUTCOME_TRAFFIC / OUTCOME_ENGAGEMENT / OUTCOME_LEADS / OUTCOME_SALES` |
| `ad_type` | — | string | `"image"` | `image / video` |
| `creative_image_url` | ✓ if image | string | `null` | ⚠️ NOT `creative_id`; must be a public image URL |
| `creative_video_url` | ✓ if video | string | `null` | |
| `creative_message` | — | string | `""` | ad text body |
| `cta_type` | — | string | `"LEARN_MORE"` | `LEARN_MORE / SHOP_NOW / SIGN_UP / BOOK_TRAVEL / DOWNLOAD / CONTACT_US / SUBSCRIBE / GET_OFFER` |
| `cta_url` | — | string | `""` | ⚠️ NOT `landing_url` |
| `daily_budget` | — | float | `5.0` | USD, min `1.0` |
| `target_age_min` | — | int | `18` | ⚠️ flat, not `targeting.age_min` |
| `target_age_max` | — | int | `65` | |
| `target_genders` | — | list[int] | `[0]` | `[0]=all, [1]=male, [2]=female` (int, not string) |
| `target_countries` | — | list[string] | `["TW"]` | ISO country codes |
| `placements` | — | list[string] | `["facebook","instagram"]` | |
| `schedule_start` | — | ISO datetime | `null` | `null` = start now |
| `schedule_end` | — | ISO datetime | `null` | `null` = no end |

**Do NOT pass `optimization_goal`** — backend derives it from `campaign_objective`:

| `campaign_objective` | auto-mapped `optimization_goal` |
|---------------------|--------------------------------|
| `OUTCOME_AWARENESS` | `REACH` |
| `OUTCOME_TRAFFIC` | `LINK_CLICKS` |
| `OUTCOME_ENGAGEMENT` | `POST_ENGAGEMENT` |
| `OUTCOME_LEADS` | `LEAD_GENERATION` |
| `OUTCOME_SALES` | `OFFSITE_CONVERSIONS` |

Response (`AdCampaignResponse`):
`{ id, social_account_id, campaign_name, campaign_objective, ad_type, status, daily_budget, cta_type, cta_url, platform_campaign_id, platform_adset_id, platform_ad_id, platform_creative_id, platform_insights, error_message, created_at }`

---

## Ad Creative Generation — `landing_ai_mcp`

### `generate_ad`

MCP: `generate_ad(user_token, session_id, data_json)` where `data_json`
is a JSON string like `{"platform": "meta", "ta_group_id": "ta_1"}`.

Returns: `{ project_id, status: "processing" }`

### `get_ad_result`

MCP: `get_ad_result(user_token, session_id, project_id)`.

When `status == "completed"`, returns: `{ status, image_url, video_url, ... }`.

Feed `image_url` directly into `create_ad_campaign.creative_image_url`
(do NOT look for a `creative_id` field — there isn't one).

---

## Error response cheatsheet

| HTTP | body hint | Root cause | What the LLM should do |
|:----:|-----------|------------|------------------------|
| `401` | `Invalid token` | AI token expired OR never exchanged | re-exchange `sc_live_*` via `/auth/ai-token/exchange` |
| `403` | `SCOPE_FORBIDDEN` | `scope=ai_agent` hit a sensitive endpoint | ask user to operate on salecraft.ai directly |
| `404` | `Not Found` on `/auth/ai-token/*` | backend has `AI_TOKEN_ENABLED=false` | stop, tell user the feature is under maintenance |
| `422` | `field required: <name>` | param name drift — compare to this file | fix the param name, do NOT invent fields |
| `422` | `value is not a valid <enum>` | enum value wrong (e.g. `TRAFFIC` instead of `OUTCOME_TRAFFIC`) | use exact enum per this file |
| `429` | `Monthly publishing limit reached` | user hit `max_social_posts_per_month` | tell user to wait until the 1st of next month |
| `400` | `No linked Facebook Page` | ads require FB Page connection on the account | tell user to bind a FB Page on salecraft.ai/marketingx |
| `500` | `column ... does not exist` | DB migration drift (backend bug) | retry after ~60s (auto-migration runs on next deploy); escalate to zereo@connact.ai if persistent |

---

## Known platform / transport quirks

### Unicode in `data_json` strings — narrow scope

Empirically verified (2026-04-18):

- ✅ **Clean round-trip** when called directly against REST endpoints.
  `session_name`, `update_stripe_texts` headline/subheadline, every
  `publish_post` caption, every ad `creative_message` — no corruption.
  Even `\uXXXX`-escaped input reaches the backend and DB identically.
- ⚠️ **Double-decode only observed on `regenerate_stripe.user_feedback`**
  (e.g. `娘` → `岣`). The value reaches the DB clean, but the prompt
  text fed to Gemini for stripe regeneration has been seen mangled —
  the corruption lives between "stored feedback text" and "prompt
  assembled for the regeneration agent", not in the MCP transport or
  REST body parsing. Rewording the feedback in English / ASCII sidesteps
  the bug.

**What this means for the LLM**:
- Don't worry about Unicode for the publish / ads / session-update paths
  — raw CJK in `data_json` is fine.
- For `regenerate_stripe`: prefer ASCII feedback, or keep CJK feedback
  short and explicit; if the regenerated stripe comes back with garbled
  text, retry with an English-worded feedback and translate back in
  `update_stripe_texts`.
- If you see `岣 / 嬤 / 娃` or similar home-radical garbled characters
  in a regenerated stripe result, that's the known double-decode — not
  a model hallucination.

### TikTok sandbox mode (pre-App-Review)

- All posts forced to `SELF_ONLY` by backend; `platform_permalink` comes
  back as empty string — this is correct, not a bug.
- Full details in `skills/publish-social/SKILL.md` → "TikTok Publishing".

### Async / polling budget

Most long-running jobs (LP generation, reels, ad creative) return
`{ project_id, status: "processing" }` and need polling. Recommended
cadence:

| Job | Interval | Max attempts per turn | Typical total |
|-----|---------:|----------------------:|--------------:|
| `get_ta_statuses` (LP) | 30s | 60 (30 min) | 5–15 min |
| `get_ad_result` (quick ad) | 30s | 10 (5 min) | 2–5 min |
| `get_carousel_result` | 30s | 20 (10 min) | 5–8 min |
| `get_reel_result` | 30s | 40 (20 min) | 5–15 min |
| `get_post_detail` (publish) | 5s | 12 (1 min) | 10–30 s |

If you exhaust the per-turn budget, **hand control back to the user**
with a progress summary; do not keep polling silently.
