# SaleCraft

**An AI marketing consultant for physical products** — primarily a **Claude Code plugin**, also usable as a markdown instruction set in any internet-connected LLM. Free consultation + paid execution (Landing Pages, Reels, social posts, ads).

## What Is SaleCraft, exactly?

Structurally this repo is a **Claude Code plugin** — `.claude-plugin/plugin.json` manifest, `commands/` slash commands (`/salecraft`, `/salecraft-create`, `/salecraft-publish`…), `skills/` Anthropic-format skill packs, root `CLAUDE.md` for long-term context. That's the **Tier 1** experience: install once in Claude Code, get auto-loaded slash commands, skills, and (optionally) the SaleCraft MCP server connection.

But because every command, skill, and pattern is plain markdown, **other LLMs can also read this repo as instructions** and follow the playbook. They just won't get auto-registered slash commands. That's the **Tier 2** experience.

### Two tiers of usage

| Tier | Where | Slash commands? | Skills auto-load? | Paid execution? |
|------|-------|-----------------|-------------------|-----------------|
| **1 — Native (recommended)** | Claude Code, Claude Desktop with plugin support | ✅ `/salecraft-create` etc. | ✅ | ✅ via MCP |
| **2 — Read-as-instructions** | Claude.ai web, ChatGPT, Gemini, Cursor, Cline, Perplexity… | ❌ (just markdown) | ❌ (LLM picks per request) | ✅ if LLM has Code Execution / fetch / MCP — see capability ladder in `CLAUDE.md` |

The free consultation works in both tiers and in 100% of LLMs (it's pure conversation — no backend needed).

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

## How To Use

### Tier 1 — Claude Code (recommended)

Install via Claude Code's plugin system:

```
/plugin install https://github.com/connactai/Salecraft-Plugin
```

After install:
- `/salecraft` — main menu
- `/salecraft-strategy`, `/salecraft-engage`, `/salecraft-retain`, `/salecraft-audit` — free consultation commands
- `/salecraft-create`, `/salecraft-edit`, `/salecraft-publish`, `/salecraft-reels` — paid execution commands
- Skills auto-load when invoked
- For paid actions, Claude Code will route through MCP if you have `landing_ai_mcp` connected, or direct REST otherwise

### Tier 2 — Any other LLM (Claude.ai, ChatGPT, Gemini, Cursor, Cline…)

You can't "install" the plugin into these — they don't have a plugin system that understands `.claude-plugin/`. But you can paste the GitHub URL and the LLM will read the repo as instructions:

```
https://github.com/connactai/Salecraft-Plugin
```

What works in this mode:
- **Free consultation (100%)**: any LLM that can read URLs can follow the saleskit / strategy / engage / retain / audit skills as a conversation guide.
- **Paid execution (depends on LLM capability)**: see the **Capability ladder** at the top of `CLAUDE.md`. Summary:
  - **Claude.ai** (with Artifacts/computer use), **Cursor**, **Cline**, **Claude Desktop + MCP** → executes natively
  - **ChatGPT Plus with Code Interpreter** → write a Python script with `requests` and run it; tell ChatGPT "use Code Interpreter to actually run this, don't just describe"
  - **Gemini Pro with Code Execution** → similar — tell it to use code execution
  - **Free ChatGPT / Gemini with browsing only** → free consultation works; for paid actions, switch to one of the above
- **Slash commands like `/salecraft-create`** are NOT registered in Tier 2 — they're just markdown filenames. Use natural language: "do a landing page for my product".

### Login flow (AI Token — no email/password, ever)

For paid actions only:

1. The LLM hands you a URL: **`https://salecraft.ai/{locale}/marketingx`** (e.g. `zh-TW`, `en`)
2. You log in (Email or Google one-click) and click **「複製 AI 登入 Token」 / "Copy AI Login Token"**
3. You paste the `sc_live_…` token back into the chat
4. The LLM exchanges it (`POST /auth/ai-token/exchange`) for a session-scoped access token, then executes the paid action and returns the real result (LP URL, post URL, ad campaign ID, etc.)

**You never give the LLM an email or password.** The AI Token has `scope: ai_agent`, expires in 12 hours, and can be revoked any time on the marketingx page.

### Optional: connect the MCP server

For Tier 1 native execution (or any MCP-aware client), add the SaleCraft MCP server:

```json
{
  "mcpServers": {
    "Service System Deep Research": {
      "type": "sse",
      "url": "https://service-system-staging-s6ykq3ylca-de.a.run.app/mcp/sse"
    }
  }
}
```

Without this, Claude Code falls back to direct REST (works equally well; pattern documented in `lib/rest-api-direct.md`).

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
- **15+ locales** — en, zh-TW, zh-CN, ja, ko, vi, fr, th, es, pt, ar (RTL), de, id, ms, hi
- **AI Landing Pages** — 30-minute turnaround, 4-stage pipeline
- **Social publishing** — IG, FB, TikTok one-click posting
- **Ad campaigns** — Meta + Google Ads creation
- **Brand audit** — Diagnose what's missing before spending
- **Transparent pricing** — AI always tells you the cost before acting
- **Any platform** — Works on ChatGPT, Claude, Gemini, Kimi, GLM, OpenClaw, and any AI platform

## Architecture

```
salecraft-plugin/
├── CLAUDE.md              # Core AI instructions (read by any AI platform)
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

## Support

Having trouble? Need help getting started?

- **Email**: zereo@connact.ai
- **GitHub Issues**: [github.com/connactai/Salecraft-Plugin/issues](https://github.com/connactai/Salecraft-Plugin/issues)

## Platform Compatibility

SaleCraft works on **any AI platform** that supports MCP (Model Context Protocol):

- ChatGPT (with MCP plugin support)
- Claude (Claude Code, Claude.ai, Claude Desktop)
- Gemini
- Kimi
- GLM
- OpenClaw
- Any other MCP-compatible AI assistant

**You do NOT need to install any specific AI tool.** Just connect the MCP server to your preferred AI platform and start chatting.

## License

MIT
