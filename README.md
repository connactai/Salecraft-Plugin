# MarketingX

A Claude Code plugin for end-to-end marketing automation — Landing Page generation, homepage building, and ad publishing via MCP orchestration.

## Overview

MarketingX enables any Claude Code instance to create professional marketing landing pages, build website homepages, and run Meta/Google ad campaigns through natural conversation. It orchestrates **200+ MCP tools** across `landing_ai_mcp` and `zereo_social_mcp` backend services.

## Installation

```bash
# Via Claude Code plugin system
claude plugin add marketingx
```

Or clone manually:
```bash
git clone https://github.com/MarinChen99097/marketingx.plugin.git
```

## Requirements

- **Claude Code** with MCP support
- **Service System Deep Research** MCP connected (provides access to `landing_ai_mcp` and `zereo_social_mcp`)
- User account on the Landing AI platform

## Skills (9)

MarketingX provides 9 skills covering the full marketing workflow:

| Phase | Skill | What Happens |
|-------|-------|-------------|
| 1 | `brand-onboard` | Verify brand assets — logo, product images, description |
| 2 | `audience-target` | AI-suggested target audiences, credit estimation |
| 3 | `generate-landing` | AI pipeline generates visual LP stripes (Strategist → Architect → Factory) |
| 4 | `edit-landing` | Stripe-level editing — text, images, overlays, red-box annotations |
| 5 | `homepage-builder` | Build a deployable website homepage embedding LP content |
| 6a | `publish-social` | Post to connected social platforms (TikTok, Instagram, Facebook) |
| 6b | `publish-ads` | Create Meta / Google ad campaigns |
| + | `i18n-adapt` | Adapt for 10 locales including RTL Arabic |
| + | `research-market` | Market research via 7+ social MCPs |

## Commands (6)

| Command | Description |
|---------|-------------|
| `/mx` | Main menu — show all capabilities |
| `/mx-create` | Full creation flow (brand → TA → LP → edit → homepage) |
| `/mx-edit` | Edit an existing landing page |
| `/mx-homepage` | Build homepage from existing LP |
| `/mx-publish` | Social posting + ad campaigns |
| `/mx-status` | Check credits / session status |

## Features

- **16:9 + 9:16** landing pages with adaptive display
- **10 locales**: en, zh-TW, ja, ko, vi, fr, th, es, pt, ar (RTL)
- **Meta + Google Ads** one-stop campaign creation
- **4-layer cultural adaptation** (text, visual, interaction, cognitive)
- **Homepage templates**: single-product, multi-product, campaign
- **Scrollable phone mockup** for LP preview embedding
- **Red-box annotations** for precise visual editing (multi-box per stripe)
- **3-step regeneration**: trigger → poll → confirm
- **Batch edits**: combine all changes into one call per stripe
- **MCP file upload**: signed URL workflow (no multipart needed)
- **Spokesperson management**: user photos as LP spokesperson
- **Industry-specific image fields**: 50+ fields across 10 industries, auto-selected by category
- **Multi-LP strategy**: generate Overview + Projects + Experience LPs for personal brands
- **Deep discovery**: structured info gathering before generation (personal brand & product flows)
- **SEO**: JSON-LD, Open Graph, hreflang, semantic HTML

## Architecture

```
marketingx.plugin/
├── .claude-plugin/
│   └── plugin.json        # Plugin metadata
├── CLAUDE.md              # AI reads this to understand everything
├── skills/                # 9 skills (SKILL.md each)
│   ├── brand-onboard/
│   ├── audience-target/
│   ├── generate-landing/
│   ├── edit-landing/
│   ├── homepage-builder/
│   ├── publish-social/
│   ├── publish-ads/
│   ├── i18n-adapt/
│   └── research-market/
├── commands/              # 6 slash commands
├── prompts/               # System context + workflow docs
├── templates/             # Homepage HTML templates + 10-locale i18n
├── lib/                   # MCP patterns, credit calc, ad specs
└── examples/              # Sample homepage output
```

## MCP Servers

### Core (required)

| Server | Tools | Purpose |
|--------|-------|---------|
| `landing_ai_mcp` | 245 | LP generation, editing, brand management, export |
| `zereo_social_mcp` | 43 | Social publishing, ad campaigns, QR codes |

### Research (optional)

| Server | Key Tools | Purpose |
|--------|-----------|---------|
| `google_trends_mcp` | `get_trending_terms`, `get_news_by_keyword` | Trend research |
| `x_mcp` | `x_search_recent`, `x_get_user` | Twitter/X sentiment |
| `reddit_mcp` | `reddit_search_posts`, `reddit_trending_posts` | Community insights |
| `tiktok_mcp` | `tiktok_search_videos` | Gen-Z trends |
| `youtube_mcp` | `youtube_search_videos` | Video content |
| `linkedin_mcp` | `linkedin_search_companies` | B2B audience |
| `meta_mcp` | `instagram_search_hashtags` | Social engagement |

> Research MCPs use **prefixed tool names** (e.g., `x_search_recent` not `search_recent`).

## Key Technical Notes

- **MCP Proxy**: All tool calls route through `mcp__claude_ai_Service_System_Deep_Research__mcp_tool_call`
- **Auth**: `login(email, password)` returns `access_token` only (no refresh_token). Re-login on 401.
- **LP URLs**: `https://landingai.info/{locale}/landing-page?id={campaign_id}`
- **Text Updates**: `update_stripe_texts` uses `{"index": N, "headline": "..."}` format (not stripe_idx/text_key)
- **Regeneration**: 3-step flow — `regenerate_stripe` → `get_stripe_regen_status` → `complete_regeneration`
- **Overlay/Soft Edge**: `set_stripe_overlay` and `set_stripe_soft_edge` require `enabled: bool` parameter
- **File Upload**: `get_asset_upload_url` → `curl -T file <signed_url>` → use `public_url` in MCP calls
- **Wizard Images**: product/logo → `wizard_shared_files`, industry-specific → `wizard_shared_data`, spokesperson → `create_spokesperson`
- **Industry Category**: must be English (`person`, `software`, `cosmetics`, etc.) — Chinese values cause validation error
- **Register**: `register(email, password, full_name)` for new users

## Tested

56 MCP tools tested across all 9 skills. 47/48 testable passed (98%).
See `TEST_RESULTS.md` for full results. See `KNOWN_ISSUES.md` for open issues.

## License

MIT
