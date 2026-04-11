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

## Authentication

All MCP calls require a JWT token obtained via:

```
mcp_tool_call(server_name="landing_ai_mcp", tool_name="login", arguments={"email": "...", "password": "..."})
```

- Returns: `{ "access_token": "eyJ...", "refresh_token": "..." }`
- Pass `user_token` parameter in all subsequent calls
- Refresh via: `mcp_tool_call(server_name="landing_ai_mcp", tool_name="refresh_tokens", arguments={"refresh_token": "..."})`
- **Test account**: `user@example.com` / `123` (development only)

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

### Authentication
```
login(email: str, password: str) -> { access_token, token_type: "bearer" }
# NOTE: No refresh_token returned. Token expires in ~12 hours. Re-login when 401.
```

### Brand
```
create_brand(user_token, name, description?, industry?, primary_color?) -> brand
get_brand(user_token, brand_id) -> brand_detail
list_brands(user_token) -> brands[]
brand_gap_analysis(user_token, brand_id) -> { missing_assets, readiness_score }
analyze_brand_url(user_token, url) -> extracted_brand_info
upload_brand_asset(user_token, brand_id, asset_type, file_url) -> asset
```

### Session / Generation
```
create_session(user_token, session_name, brand_name?, product_name?, base_description?, conversation_id?) -> session
update_session(user_token, session_id, ...) -> updated_session
get_session(user_token, session_id) -> { status, stripes[], ... }
generate_ta_options(user_token, brand_name, description, industry_category, product_name?, ..., ui_locale="zh-TW") -> ta_options[]
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
regenerate_stripe(user_token, campaign_id, stripe_idx: int) -> regenerated_stripe
update_stripe_text_styling(user_token, campaign_id, stripe_idx, styling_json) -> updated
update_stripe_background(user_token, campaign_id, stripe_idx, ...) -> updated
update_image_layers(user_token, campaign_id, stripe_idx, ...) -> updated
set_stripe_overlay(user_token, campaign_id, stripe_idx, overlay_json) -> updated
set_stripe_soft_edge(user_token, campaign_id, stripe_idx, ...) -> updated
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

### Publishing
```
list_accounts(user_token) -> accounts[]
get_account_capability(user_token, account_id) -> { can_advertise, permissions }
import_from_session(user_token, session_id) -> imported_content
validate_content_media(user_token, ...) -> validation_result
publish_post(user_token, data_json) -> post_result
publish_multi(user_token, targets_json) -> results[]
schedule_post(user_token, data_json) -> scheduled
suggest_schedule(user_token) -> optimal_times[]
generate_qr(user_token, url, style?) -> qr_image_url
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
generate_ad(user_token, session_id, data_json) -> { project_id, ... }
# data_json example: '{"platform": "meta"}' — platform goes INSIDE data_json
get_ad_result(user_token, session_id, project_id) -> { status, creative, ... }
# NOTE: Uses session_id + project_id, NOT task_id
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

## Security Rules

- **Never hardcode API keys or tokens** — always use `user_token` parameter
- **Never modify real user data** — use test account `user@example.com` for development
- **Never expose JWT tokens** in generated HTML or client-side code
- **All MCP calls** go through the Service System proxy — never call backend APIs directly

## Credit System

- Each LP generation costs credits (varies by stripe count and model)
- Check balance via `get_me(user_token)` before generation
- `generate_ta_options` returns estimated cost per TA group
- Credits are deducted on `generate_session`, auto-refunded on interruption
