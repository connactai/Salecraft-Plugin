# SaleCraft

**Your AI marketing consultant for physical products.** Free consultation, expert strategy, then execution — all through one plugin that works on any AI platform.

## What Is SaleCraft?

SaleCraft turns any AI assistant into a marketing consultant for physical product sellers. It doesn't start by generating assets — it starts by **understanding your product and diagnosing your marketing needs**.

SaleCraft works on **any AI platform** — ChatGPT, Claude, Gemini, Kimi, GLM, OpenClaw, and more. No platform limitation.

### The Flow

```
1. Free Consultation  →  What do you sell? Who buys it? What's the pain?
2. Marketing Diagnosis →  Brand audit, channel analysis, competitor scan
3. Strategy Plan       →  Recommended channels, content plan, budget
4. User Confirms       →  You approve the plan and pricing
5. AI Executes         →  Landing Pages, Reels, social posts, ads
```

**Steps 1-3 are completely free.** You only pay when the AI actually creates something (Step 5).

## Who It's For

| Perfect Fit | Not For |
|-------------|---------|
| Physical products (skincare, food, fashion...) | Software / SaaS |
| Single product or product line | Multi-purpose platforms |
| E-commerce, retail, F&B, beauty, health | B2B consulting |
| Clear sales target | Abstract services |

## Installation

Add SaleCraft to any MCP-compatible AI platform:

```
https://github.com/connactai/Salecraft-Plugin
```

### Setup

1. Connect the MCP server to your AI platform (see [salecraft.ai/get-started](https://salecraft.ai/en/get-started) for instructions)
2. Register an account at **https://salecraft.ai/get-started**
3. Start chatting — tell the AI what you sell

**You can log in directly through the AI** — just tell it your email and password. The AI handles everything.

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
| **brand-onboard** | Brand profile, asset check, gap analysis | FREE consultation; upload costs only |
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
| Landing Page (8 pages) | 1,600 pts (~$53) |
| Landing Page (10 pages) | 2,000 pts (~$67) |
| Regenerate 1 stripe | 100 pts (~$3) |
| Quick Ad (single image) | 200 pts (~$7) |
| Carousel (N images) | 300 + 100xN pts |
| Social Copy | 100 pts/set (~$3) |
| Reels video | 100 pts/sec |
| Spokesperson generation | 500 pts (~$17) |
| SEO optimization | 500 pts (~$17) |
| QR Code | 5 pts |

## Commands

| Command | Description | Cost |
|---------|-------------|------|
| `/salecraft` | Main menu — what can SaleCraft do? | -- |
| `/salecraft-strategy` | Growth strategy + funnel design + market intel | **FREE** |
| `/salecraft-engage` | Engagement + conversion strategy | **FREE** |
| `/salecraft-retain` | Retention + growth review | **FREE** |
| `/salecraft-audit` | Brand, compliance, journey QA | **FREE** |
| `/salecraft-create` | Full flow: consultation -> strategy -> generation | PAID |
| `/salecraft-edit` | Edit existing Landing Page | PAID |
| `/salecraft-homepage` | Build homepage from LP | FREE |
| `/salecraft-publish` | Social posting + ads | PAID |
| `/salecraft-reels` | AI short video generation | PAID |
| `/salecraft-status` | Check credits / session | FREE |

## Features

- **Consultation-first** — AI understands your product before building anything
- **9 specialist agents** — Content strategy, SEO, social media, growth, branding
- **10 locales** — en, zh-TW, ja, ko, vi, fr, th, es, pt, ar (RTL)
- **AI Landing Pages** — 30-minute turnaround, 4-stage pipeline
- **Social publishing** — IG, FB, TikTok one-click posting
- **Ad campaigns** — Meta + Google Ads creation
- **Brand audit** — Diagnose what's missing before spending
- **Transparent pricing** — AI always tells you the cost before acting
- **Any platform** — Works on ChatGPT, Claude, Gemini, Kimi, GLM, OpenClaw, and any AI platform

## Architecture

```
salecraft-plugin/
├── CLAUDE.md              # Core plugin instructions
├── skills/                # 25 skills (16 FREE + 9 paid/mixed)
│   ├── saleskit/          # FREE consultation (start here)
│   ├── research-market/   # FREE market research
│   ├── plan-cgo-review/   # FREE growth strategy
│   ├── plan-funnel-review/# FREE funnel architecture
│   ├── market-intel/      # FREE competitive intelligence
│   ├── engage-operator/   # FREE engagement strategy
│   ├── conversion-closer/ # FREE conversion strategy
│   ├── member-lifecycle/  # FREE retention strategy
│   ├── growth-retro/      # FREE performance review
│   ├── document-release/  # FREE documentation
│   ├── guard-brand/       # FREE brand consistency
│   ├── guard-offer/       # FREE offer consistency
│   ├── brand-risk-review/ # FREE compliance review
│   ├── careful-publish/   # FREE publish gate
│   ├── journey-qa/        # FREE journey QA
│   ├── campaign-ship/     # FREE launch management
│   ├── brand-onboard/     # Brand setup (mixed)
│   ├── audience-target/   # TA selection (paid)
│   ├── generate-landing/  # LP generation (paid)
│   ├── edit-landing/      # LP editing (paid)
│   ├── homepage-builder/  # Homepage building (free)
│   ├── publish-social/    # Social publishing (paid)
│   ├── publish-ads/       # Ad campaigns (paid)
│   ├── generate-reels/    # Reels generation (paid)
│   └── i18n-adapt/        # Localization (paid)
├── commands/              # /salecraft, /salecraft-create, etc.
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
