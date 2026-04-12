# MarketingX Plugin — CLAUDE.md

> **Any Claude Code instance reading this file has everything needed to create marketing landing pages, build homepages, and publish ads end-to-end.**

## What This Plugin Does

MarketingX is a Claude Code plugin that orchestrates 200+ MCP tools across two backend services to deliver a complete marketing workflow:

1. **Brand Onboarding** — Verify sales intent, check brand assets (images, copy, knowledge)
2. **Audience Targeting** — AI-suggested target audiences, page count estimation, credit cost
3. **Landing Page Generation** — AI pipeline: Strategist → Architect → Factory → Stripe Reflector
4. **Post-Generation Editing** — Stripe-level text, image, overlay, crop, regenerate
5. **Homepage Building** — Embed LP stripes into a deployable website homepage
6. **Social Publishing** — Multi-platform posting (Meta, TikTok, etc.)
7. **Ad Campaigns** — Meta / Google Ads one-stop creation and management
8. **i18n Adaptation** — 10 locales, RTL support, 4-layer cultural adaptation
9. **Market Research** — Optional trend/competitor analysis via 7+ social MCPs

## MCP Dependency (REQUIRED)

This plugin requires the **Service System Deep Research** MCP to be connected. All tool calls route through:

```
mcp__claude_ai_Service_System_Deep_Research__mcp_tool_call(
  server_name = "landing_ai_mcp" | "zereo_social_mcp" | ...,
  tool_name   = "<tool>",
  arguments   = { ... }
)
```

If this MCP is not available, the plugin cannot function.

## First-Time Setup (Cold Start)

When a user first invokes this plugin:

1. **Check MCP**: Verify `landing_ai_mcp` is reachable via the Service System Deep Research proxy
2. **Account**: Ask if they have an account → Login or Register
3. **Start**: `/mx` for menu, or `/mx-create` for full flow

## Authentication

### Login (existing user)
```
mcp_tool_call("landing_ai_mcp", "login", {"email": "...", "password": "..."})
→ { "access_token": "eyJ...", "token_type": "bearer" }
```

### Register (new user)
```
mcp_tool_call("landing_ai_mcp", "register", {"email": "...", "password": "...", "full_name": "..."})
→ creates account + returns access_token
```

### Google OAuth
```
mcp_tool_call("landing_ai_mcp", "google_auth", {"credential": "<google_id_token>"})
```

### Auth Notes
- Pass `user_token` in ALL subsequent calls
- Token expires ~12 hours. On 401, re-call `login` (no refresh_token)
- **Test account**: Use your own Landing AI account credentials

## Available Commands

| Command | Purpose | Workflow |
|---------|---------|----------|
| `/mx` | Main menu — show capabilities | — |
| `/mx-create` | Full creation flow | Phase 1→5 (brand → TA → generate → edit → homepage) |
| `/mx-edit` | Edit existing LP | Phase 4 only |
| `/mx-homepage` | Build homepage from existing LP | Phase 5 only |
| `/mx-publish` | Publish + run ads | Phase 6a + 6b |
| `/mx-status` | Check session status / credits | — |

## Available Skills (9)

| Skill | Phase | Core MCP | Key Tools |
|-------|-------|----------|-----------|
| `brand-onboard` | 1 | `landing_ai_mcp` | `login`, `list_brands`, `brand_gap_analysis`, `upload_brand_asset` |
| `audience-target` | 2 | `landing_ai_mcp` | `generate_ta_options`, `get_me`, `get_generation_settings` |
| `generate-landing` | 3 | `landing_ai_mcp` | `create_session`, `generate_session`, `get_session`, `list_stripes` |
| `edit-landing` | 4 | `landing_ai_mcp` | `update_stripe_texts`, `regenerate_stripe`, `crop_stripe`, `undo_stripe` |
| `homepage-builder` | 5 | `landing_ai_mcp` | `export_html`, `download_stripe`, `get_landing_page` |
| `publish-social` | 6a | `zereo_social_mcp` | `list_accounts`, `publish_multi`, `suggest_schedule` |
| `publish-ads` | 6b | `zereo_social_mcp` + `landing_ai_mcp` | `create_ad_campaign`, `generate_ad`, `get_ad_objectives` |
| `i18n-adapt` | any | `landing_ai_mcp` | `update_stripe_texts`, `update_stripe_text_styling`, `regenerate_stripe` |
| `research-market` | pre | 7+ social MCPs | `google_trends_mcp`, `x_mcp`, `reddit_mcp`, etc. |

## Core MCP Servers

### landing_ai_mcp (160+ tools)

| Module | Tools | Purpose |
|--------|-------|---------|
| Session Wizard | 32 | Create sessions, generate LPs, TA options |
| Landing Page Editor | 49 | Edit stripes (text, image, overlay, crop, reorder, export) |
| Brand Management | 29 | Brand CRUD, gap analysis, asset upload |
| Conversations | 20 | Message-based interaction, asset classification |
| Reels | 26 | Short video generation pipeline |
| Content | 45 | URL scraping, PDF import, SEO, QR, campaigns |
| Workspace | 10 | Project management |
| Todos | 14 | Task tracking, AI plan |

### zereo_social_mcp (43 tools)

| Module | Tools | Purpose |
|--------|-------|---------|
| Social Accounts | 10 | Platform connection management (Meta/TikTok OAuth) |
| Content Publishing | 8 | Post/schedule to multiple platforms |
| Content Library | 10 | Import, validate, manage generated content |
| Ad Campaigns | 11 | Create/manage Meta & Google ad campaigns |
| QR Code | 3 | Generate styled QR codes |
| AI Scheduling | 2 | Optimal posting time suggestions |

## Key Tool Signatures

### Authentication & Account Management
```
login(email: str, password: str) -> { access_token, token_type: "bearer" }
register(email: str, password: str, full_name: str) -> { user, access_token }
google_auth(credential: str) -> { access_token, token_type }
refresh_tokens(refresh_token: str) -> { access_token }
forgot_password(email: str) -> { message }
reset_password(token: str, new_password: str) -> { message }
verify_email(token: str) -> { message }
resend_verification(email: str) -> { message }
get_me(user_token) -> { id, email, full_name, credits, tier, is_admin, ... }
update_me(user_token, data_json) -> updated_user
update_user_settings(user_token, data_json) -> updated_settings
logout(user_token) -> { message }
delete_account(user_token) -> { message }
cancel_deletion(user_token) -> { message }
# NOTE: login returns access_token only (no refresh_token). Token expires ~12h. Re-login on 401.
```

### Brand
```
create_brand(user_token, name, description?, industry?, primary_color?) -> brand
get_brand(user_token, brand_id) -> brand_detail
list_brands(user_token) -> brands[]
brand_gap_analysis(user_token, brand_id) -> { missing_assets, readiness_score }
analyze_brand_url(user_token, url) -> extracted_brand_info
upload_brand_asset(user_token, brand_id, asset_type, file_url) -> asset
get_asset_upload_url(user_token, brand_id, filename, asset_type?, content_type?) -> { upload_url, gcs_path, public_url }
# MCP-compatible file upload! Flow:
# 1. Call get_asset_upload_url → get signed PUT URL
# 2. bash: curl -X PUT -H "Content-Type: image/jpeg" -T /path/to/photo.jpg "{upload_url}"
# 3. Use public_url in create_spokesperson, regenerate_stripe reference_image_urls_json, etc.
create_brand_asset(user_token, brand_id, data_json) -> asset
list_brand_assets(user_token, brand_id) -> assets[]
delete_brand_asset(user_token, brand_id, asset_id) -> { message }
create_spokesperson(user_token, brand_id, name, description, photo_urls[]) -> spokesperson
# ⚠️ create_spokesperson does NOT persist photo_urls! After creating, MUST call:
update_spokesperson(user_token, brand_id, sp_id, data_json) -> updated
# data_json: {"photo_urls": ["url"], "name": "...", "description": "..."}
# This is the ONLY reliable way to set spokesperson photos.
list_spokespersons(user_token, brand_id) -> spokespersons[]
add_spokesperson_photos(user_token, brand_id, sp_id, photo_urls_json) -> ❌ multipart stub
# Use update_spokesperson instead for setting photo_urls
delete_spokesperson(user_token, brand_id, sp_id) -> { message }
# NOTE: parameter name is "sp_id" not "spokesperson_id"
# Wizard session images: view via get_session, add/delete via update_session
# Delete = pass array WITHOUT the URL to remove (arrays are replaced, not appended)
```

### Session / Generation
```
create_session(user_token, session_name, brand_name?, product_name?, base_description?, conversation_id?) -> session
update_session(user_token, session_id, ...) -> updated_session
get_session(user_token, session_id) -> { status, stripes[], ... }
generate_ta_options(user_token, brand_name, description, industry_category, product_name?, ..., ui_locale="zh-TW") -> ta_options[]
# industry_category MUST be English: general, food, healthy_meals, restaurant, desserts, gift_box,
# medical_aesthetics, person, consultant, film, property, private_kitchen, supplements, cosmetics,
# biotech, software, electronics, home_appliances, education, fashion, sports, travel, finance,
# real_estate, automotive. Use "person" for personal brands, "software" for tech.
get_generation_settings(user_token) -> settings
generate_session(user_token, session_id, ta_group_ids_json, requested_stripe_count?) -> { status: "processing", project_ids[] }
# IMPORTANT: Before calling generate_session, you MUST:
# 1. Call update_session with wizard_ta_groups data (from generate_ta_options output)
# 2. Call get_ta_statuses to get the auto-assigned ta_group_id values
# 3. Pass those IDs as ta_group_ids_json: '["ta_1"]'
get_me(user_token) -> { credits_remaining, tier, ... }
```

### Landing Page Viewing
```
list_stripes(user_token, session_id) -> stripes[]
get_stripe_detail(user_token, campaign_id, stripe_idx) -> stripe_detail
get_landing_page(user_token, campaign_id) -> landing_page_config
```

### Landing Page Editing
```
update_stripe_texts(user_token, campaign_id, updates_json) -> updated_stripe
# updates_json format: '[{"index": 0, "headline": "New Text", "subheadline": "..."}]'
# IMPORTANT: Use "index" (not "stripe_idx"), and field names directly (not "text_key"/"new_text")
regenerate_stripe(user_token, campaign_id, stripe_idx, user_feedback?, mandatory_text_overrides_json?, typography_overrides_json?, rect_annotations_json?, reference_image_urls_json?) -> { is_regenerating, iteration }
get_stripe_regen_status(user_token, campaign_id, stripe_idx) -> { is_regenerating, regeneration_progress }
complete_regeneration(user_token, campaign_id, stripe_idx) -> { message }
cancel_stripe_regen(user_token, campaign_id, stripe_idx) -> { message }
# REGENERATION IS 3 STEPS: regenerate_stripe → poll get_stripe_regen_status → complete_regeneration
# complete_regeneration is MANDATORY to apply the result
update_stripe_text_styling(user_token, campaign_id, stripe_idx, styling_json) -> updated
update_stripe_background(user_token, campaign_id, stripe_idx, ...) -> updated
update_image_layers(user_token, campaign_id, stripe_idx, ...) -> updated
set_stripe_overlay(user_token, campaign_id, stripe_idx, enabled: bool, overlay_json) -> updated
set_stripe_soft_edge(user_token, campaign_id, stripe_idx, enabled: bool, soft_edge_json) -> updated
# NOTE: Both require `enabled` as a mandatory parameter (true to apply, false to remove).
crop_stripe(user_token, campaign_id, stripe_idx, crop_json) -> updated
reset_crop(user_token, campaign_id, stripe_idx) -> updated
undo_stripe(user_token, campaign_id, stripe_idx) -> previous_state
redo_stripe(user_token, campaign_id, stripe_idx) -> next_state
reorder_stripes(user_token, campaign_id, order_json) -> updated
hide_stripe(user_token, campaign_id, stripe_idx) -> updated
restore_stripe(user_token, campaign_id, stripe_idx) -> updated
update_seo(user_token, campaign_id, seo_json) -> updated
update_faq_content(user_token, campaign_id, faq_json) -> updated
```

### Landing Page Export
```
export_html(user_token, campaign_id) -> html_string
download_stripe(user_token, campaign_id, stripe_idx: int) -> { download_url, method, auth_header }
download_all_stripes(user_token, campaign_id) -> { download_url, ... }
export_landing_image(user_token, campaign_id) -> { download_url, ... }
# NOTE: download_stripe returns a URL that requires the auth_header to fetch the actual image.
```

### Social Account Connection (OAuth)
```
get_meta_auth_url(user_token, redirect_uri) -> { auth_url, state }
# User opens auth_url in browser → authorizes → Meta redirects with code
connect_meta_account(user_token, code, redirect_uri) -> connected_account
# Completes OAuth — connects Facebook page + Instagram business

get_tiktok_auth_url(user_token) -> { auth_url }
# TikTok callback is server-side — user just opens URL and authorizes
```

### Social Account Management
```
list_accounts(user_token) -> accounts[]
get_account_capability(user_token, account_id) -> { can_advertise, permissions }
recheck_capability(user_token, account_id) -> { detail }
refresh_account_token(user_token, account_id) -> refreshed_account
update_ad_account(user_token, account_id, data_json) -> updated
get_account_content(user_token, account_id) -> content_list
disconnect_account(user_token, account_id) -> confirmation (destructive!)
```

### Social Post Generation (AI)
```
social_copy(user_token, data_json) -> generated_copy
# data_json: {"conversation_id": "", "quantity": 1} — deducts credits
get_session_content(user_token, session_id) -> { stripes, social_copy }
# Returns Architect-generated social copy (caption + hashtags) if available
social_reflect(user_token, data_json) -> { reflection_score, reflection_data }
# data_json: {"post_text": "...", "image_url": "...", "brand_name": "..."}
# Score < 7 = rewrite recommended
```

### Content Publishing
```
publish_post(user_token, data_json) -> post_result
# data_json: {"social_account_id": "...", "post_type": "ig_post", "caption": "...", "image_url": "..."}
# ⚠️ Parameter name is "social_account_id" (NOT "account_id") in ALL publish tools
publish_multi(user_token, targets_json) -> results[]
# targets_json: [{"social_account_id": "...", "caption": "...", "media_ids": ["..."]}]
schedule_post(user_token, data_json) -> scheduled
cancel_post(user_token, post_id) -> confirmation
get_post_history(user_token, limit?, offset?, status_filter?) -> posts[]
get_post_detail(user_token, post_id) -> post_detail
# ⚠️ LP images (9:16) → use ig_story. ig_post requires 1:1 or 4:5 aspect ratio.
# Quick Ad images (1:1) work for ig_post.
```

### Content Library
```
list_content(user_token) -> content[]
get_content(user_token, content_id) -> content_detail
import_from_session(user_token, session_id) -> imported_content
update_content(user_token, content_id, data_json) -> updated
delete_content(user_token, content_id) -> { message }
publish_content(user_token, content_id, data_json) -> { post_id, status }
# data_json: {"social_account_id": "...", "post_type": "ig_post|ig_story|fb_post|tt_video"}
validate_content_media(user_token, content_id, target_post_type, duration_seconds?) -> validation
get_upload_signed_url(user_token, filename, content_type) -> { upload_url }
confirm_upload(user_token, data_json) -> confirmed
# list_shared_content — influencer-only, requires is_influencer=True + shared_with permission check
```

### AI Scheduling
```
suggest_schedule(user_token, content_id?, caption?, content_type?, ta_info_json?) -> optimal_times[]
parse_schedule(user_token, text) -> { schedules: [{ datetime_iso, platforms }] }
```

### QR Code
```
generate_qr(user_token, url, size?) -> { qr_url }
composite_qr(user_token, image_url, qr_target_url) -> { url } (QR overlaid on image)
generate_qr_card(user_token, url, title?) -> { url } (branded QR card)
```

### Ad Campaigns
```
get_ad_objectives(user_token) -> objectives[]
get_cta_types(user_token) -> cta_types[]
create_ad_campaign(user_token, data_json) -> campaign
list_ad_campaigns(user_token) -> campaigns[]
get_ad_campaign(user_token, campaign_id, refresh?) -> campaign_status
pause_ad_campaign(user_token, campaign_id) -> result
resume_ad_campaign(user_token, campaign_id) -> result
generate_ad(user_token, session_id, data_json) -> { project_id, status: "processing" }
# data_json MUST include: '{"platform": "meta", "ta_group_id": "ta_1"}'
# ta_group_id is REQUIRED — get it from generate_session or get_ta_statuses
get_ad_result(user_token, session_id, project_id) -> { status, creative, ... }
# NOTE: Uses session_id + project_id, NOT task_id
```

### Reel Promotion (Boost)
```
promote_reel(user_token, data_json) -> { promotion_id }
# data_json: { social_post_id, cta_type, cta_url, daily_budget, duration_days }
get_promotion_status(user_token, promotion_id) -> status
pause_promotion(user_token, promotion_id) -> result
resume_promotion(user_token, promotion_id) -> result
```

## Aspect Ratio Support

Both **16:9** (landscape) and **9:16** (portrait) are supported:

- Set via `update_session` before generation
- If user wants both, create two separate sessions
- Homepage embedding uses adaptive CSS:
  - 16:9: full-width on desktop, stacked on mobile
  - 9:16: centered column (max-width 480px) on desktop, full-width on mobile

## i18n — 10 Locales

`en`, `zh-TW`, `ja`, `ko`, `vi`, `fr`, `th`, `es`, `pt`, `ar`

- Arabic (`ar`) requires RTL: `dir="rtl"` + CSS logical properties
- Japanese: longer text (20-40%), ですます体
- Korean: 해요체, high visual polish
- Pass `ui_locale` parameter in `generate_ta_options` and session creation

## Ad Platform Specs

| Platform | Format | Size |
|----------|--------|------|
| Meta Feed | Square | 1080×1080 |
| Meta Stories/Reels | Portrait | 1080×1920 (9:16) |
| Google Display | Responsive | headlines + descriptions + images |
| Google Search | Text | headlines + descriptions + sitelinks |

## File Structure

```
marketingx.plugin/
├── CLAUDE.md              ← You are here
├── skills/                # 9 skills (SKILL.md each)
├── commands/              # 6 slash commands
├── prompts/               # BOOTSTRAP.md, WORKFLOW.md
├── templates/             # Homepage HTML templates + i18n
├── lib/                   # Reference docs (MCP patterns, credit calc, etc.)
└── examples/              # Sample outputs
```

## File Upload (Universal — works in ALL skills)

Any skill that needs to upload a local file uses this 3-step flow:

```
# 1. Get signed URL
mcp_tool_call("landing_ai_mcp", "get_asset_upload_url", {
  "user_token": token, "brand_id": brand_id,
  "filename": "photo.jpg", "asset_type": "product", "content_type": "image/jpeg"
})
→ { "upload_url": "https://storage...?X-Goog-Signature=...", "public_url": "https://storage.../photo.jpg" }

# 2. Upload via curl (bash)
curl -X PUT -H "Content-Type: image/jpeg" -T "/path/to/photo.jpg" "{upload_url}"

# 3. Use public_url in the appropriate target:
```

| Target | Where to put public_url |
|--------|------------------------|
| Wizard product images | **BOTH** `wizard_shared_data.product_images` + `wizard_shared_files.product_images` |
| Wizard logo | `wizard_shared_files.logo_image: "url"` (single string, not array) |
| Wizard evidence/certs | **BOTH** `wizard_shared_data.certification_images` + `wizard_shared_files.evidence_images` |
| Wizard LP reference | `wizard_shared_data.landing_page_images: ["url"]` |
| Wizard spokesperson | `wizard_shared_data.spokesperson_faces: ["url"]` |
| Wizard industry images | `wizard_shared_data.{field}_images: ["url"]` |
| Regeneration reference | `regenerate_stripe` → `reference_image_urls_json: '["url"]'` |
| Spokesperson entity | `create_spokesperson` → `photo_urls: ["url"]` |

⚠️ **CRITICAL**: `wizard_shared_data` = frontend UI display, `wizard_shared_files` = Factory AI reads.
Product images and evidence MUST be written to BOTH or one side won't see them.

Allowed `asset_type`: `product`, `logo`, `spokesperson`, `certification`
Allowed `content_type`: `image/jpeg`, `image/png`, `image/webp`, `image/heic`, `application/pdf`

### Inline Image Upload (user pastes image in chat)
```
upload_base64(user_token, brand_id, filename, base64_data, asset_type?, content_type?) -> { public_url }
```
Claude Code can read pasted images as base64. Use `upload_base64` to upload directly — no need to save to disk.
Max 10MB per file. Supports data URI prefix stripping (e.g., `data:image/jpeg;base64,...`).

## Landing Page Frontend URLs

Generated LPs are viewable as interactive sales pages on the Landing AI frontend:

```
https://landingai.info/{locale}/landing-page?id={campaign_id}
```

- `locale`: `zh-TW`, `en`, `ja`, `ko`, etc.
- `campaign_id`: the project UUID from `generate_session` response
- This is the **primary link** to show users — NOT the GCS image URL
- The frontend fetches data via `/landing-page/public/{campaign_id}` (no auth needed)

## Security Rules

- **Never hardcode API keys or tokens** — always use `user_token` parameter
- **Never modify real user data** — always confirm with user before destructive operations
- **Never expose JWT tokens** in generated HTML or client-side code
- **All MCP calls** go through the Service System proxy — never call backend APIs directly

## Credit System

- Each LP generation costs credits (varies by stripe count and model)
- Check balance via `get_me(user_token)` before generation
- `generate_ta_options` returns estimated cost per TA group
- Credits are deducted on `generate_session`, auto-refunded on interruption
