# SaleCraft

**Your AI marketing consultant for physical products.** Free consultation, expert strategy, then execution — all through one MCP plugin.

## What Is SaleCraft?

SaleCraft is an MCP plugin that turns any AI platform into a marketing consultant for physical product sellers. It doesn't start by generating assets — it starts by **understanding your product and diagnosing your marketing needs**.

### The Flow

```
1. 🎯 Free Consultation  →  What do you sell? Who buys it? What's the pain?
2. 📊 Marketing Diagnosis →  Brand audit, channel analysis, competitor scan
3. 📋 Strategy Plan       →  Recommended channels, content plan, budget
4. ✅ User Confirms       →  You approve the plan and pricing
5. 🏭 AI Executes         →  Landing Pages, Reels, social posts, ads
```

**Steps 1-3 are completely free.** You only pay when the AI actually creates something (Step 5).

## Who It's For

| ✅ Perfect Fit | ❌ Not For |
|---------------|-----------|
| Physical products (skincare, food, fashion...) | Software / SaaS |
| Single product or product line | Multi-purpose platforms |
| E-commerce, retail, F&B, beauty, health | B2B consulting |
| Clear sales target | Abstract services |

## Installation

Add this plugin to any MCP-compatible AI platform:

```
https://github.com/connactai/Salecraft-Plugin
```

### MCP Server Setup

SaleCraft requires a Remote MCP connection:

```json
{
  "mcpServers": {
    "Service System Deep Research": {
      "type": "sse",
      "url": "https://service-system-staging-876464738390.asia-east1.run.app/mcp/sse"
    }
  }
}
```

### Account Setup

First-time users: **https://salecraft.ai/en/get-started**

This handles registration (Google or email), social account binding (FB/IG), Google Drive access, and points top-up.

## Skills (25)

### Free — No Account Needed (AI Consultation)

| Skill | What It Does |
|-------|-------------|
| **saleskit** | Free marketing consultation — diagnose needs, recommend strategy |
| **research-market** | Market research, competitor analysis, trend scanning |
| **plan-cgo-review** | Growth strategy — expand, focus, or reduce? |
| **plan-funnel-review** | Funnel architecture — traffic to repurchase journey |
| **market-intel** | Competitive intelligence — pricing, positioning, gaps |
| **engage-operator** | DM scripts, FAQ trees, auto-reply, booking flows |
| **conversion-closer** | Objection handling, pricing framing, closing scripts |
| **member-lifecycle** | Repurchase triggers, referral programs, VIP system |
| **growth-retro** | Campaign review, KPI analysis, next sprint planning |
| **guard-brand** | Brand voice and visual consistency check |
| **guard-offer** | Price and claim consistency across touchpoints |
| **brand-risk-review** | Compliance review (medical, financial, legal) |
| **careful-publish** | Final gate for high-risk content |
| **journey-qa** | End-to-end customer journey QA testing |
| **campaign-ship** | Launch checklist and monitoring plan |
| **document-release** | Compile SOPs, playbooks, FAQ manuals |

### Paid — Account Required (AI Generation + Publishing)

| Skill | What It Does | Cost (pts) |
|-------|-------------|------------|
| **brand-onboard** | Brand profile, asset check, gap analysis | FREE consultation; MCP upload costs only |
| **audience-target** | AI target audience suggestions | 5-15 |
| **generate-landing** | AI Landing Page (4-stage pipeline) | 1,600-2,000 |
| **edit-landing** | Edit LP text, images, layout | 100/regen |
| **homepage-builder** | Build website from LP | FREE |
| **publish-social** | Generate social copy + post | 100/set |
| **publish-ads** | Meta/Google ad campaigns | depends on ad creation |
| **generate-reels** | AI short video generation | 100/sec |
| **i18n-adapt** | Adapt for 10 locales (incl. RTL Arabic) | depends on regeneration |

## Pricing

| Amount | Points |
|--------|--------|
| **$1 USD** | 30 pts |
| **$20 USD** (min) | 600 pts |

| Action | Cost |
|--------|------|
| Landing Page (8 pages × 1 TA) | 1,600 pts (~$53) |
| Landing Page (10 pages × 1 TA) | 2,000 pts (~$67) |
| Regenerate 1 stripe | 100 pts (~$3) |
| Quick Ad (single image) | 200 pts (~$7) |
| Carousel (N images) | 300 + 100×N pts |
| Social Copy | 100 pts/set (~$3) |
| Reels video | 100 pts/sec (e.g. 10s = 1,000 pts) |
| Spokesperson generation | 500 pts (~$17) |
| SEO optimization | 500 pts (~$17) |
| QR Code | 5 pts |

## Commands

| Command | Description | Cost |
|---------|-------------|------|
| `/mx` | Main menu — what can SaleCraft do? | — |
| `/mx-strategy` | Growth strategy + funnel design + market intel | **FREE** |
| `/mx-engage` | Engagement + conversion strategy | **FREE** |
| `/mx-retain` | Retention + growth review | **FREE** |
| `/mx-audit` | Brand, compliance, journey QA | **FREE** |
| `/mx-create` | Full flow: consultation → strategy → generation | PAID |
| `/mx-edit` | Edit existing Landing Page | PAID |
| `/mx-homepage` | Build homepage from LP | FREE |
| `/mx-publish` | Social posting + ads | PAID |
| `/mx-reels` | AI short video generation | PAID |
| `/mx-status` | Check credits / session | FREE |

## Features

- **Consultation-first** — AI understands your product before building anything
- **9 specialist agents** — Content strategy, SEO, social media, growth, branding
- **10 locales** — en, zh-TW, ja, ko, vi, fr, th, es, pt, ar (RTL)
- **AI Landing Pages** — 30-minute turnaround, 4-stage pipeline
- **Social publishing** — IG, FB, TikTok one-click posting
- **Ad campaigns** — Meta + Google Ads creation
- **Brand audit** — Diagnose what's missing before spending
- **Transparent pricing** — AI always tells you the cost before acting

## Architecture

```
Salecraft-Plugin/
├── CLAUDE.md              # Core plugin instructions
├── skills/                # 10 skills
│   ├── saleskit/          # FREE consultation (start here)
│   ├── brand-onboard/
│   ├── audience-target/
│   ├── generate-landing/
│   ├── edit-landing/
│   ├── homepage-builder/
│   ├── publish-social/
│   ├── publish-ads/
│   ├── i18n-adapt/
│   └── research-market/
├── commands/              # /mx, /mx-create, etc.
├── prompts/               # System context
├── templates/             # Homepage HTML templates
├── lib/                   # Reference docs
└── examples/              # Sample outputs
```

## MCP Servers

| Server | Purpose |
|--------|---------|
| `landing_ai_mcp` | LP generation, editing, brand management |
| `zereo_social_mcp` | Social publishing, ads, QR codes |
| 7+ research MCPs | Google Trends, X, Reddit, TikTok, YouTube... |

## License

MIT
