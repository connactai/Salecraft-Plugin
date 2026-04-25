---
name: brand-memory
description: |
  Automatically record and recall brand context across sessions. Every file
  the user uploads, every paid action they trigger, and every meaningful
  prompt gets saved so SaleCraft can provide personalized, context-aware
  marketing advice. The user never needs to repeat themselves — their
  Project (=brand) history follows them.

  This skill runs SILENTLY in the background. It is NOT a user-facing
  command. The ONLY user-visible trigger is when they say "記憶這些" /
  "memorize this" / "remember this" — then `/memorize` is called.

  Auto-triggers: at session start (load context), after every file upload
  (save-file with vision description), after every paid tool call (save-prompt
  with resulted_in_paid=true), after meaningful consultation exchanges
  (save-prompt), and at the 5th prompt or every 24h (compile-metadata).
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
---

# Brand Memory — Project-Scoped Context Recall (MANDATORY)

> **Project = Brand**. In SaleCraft, one user owns multiple `BrandProfile`s.
> Each brand_id is one independent marketing Project. ALL memory is scoped
> by `(user_id, brand_id)` — recall NEVER crosses Projects.

This skill runs **silently in the background** (except `/memorize`, see
Rule 1 below). The user should never see "saving to memory" or "loading
context" messages. It just makes SaleCraft smarter over time.

> **Currency**: when recording brand context that mentions prices (in
> `user_input` / `ai_response_summary` / file descriptions), **normalize
> to USD (`$`)** if possible — never store NT$ / EUR / £ / ¥ / 円 / 人民幣
> / KRW / THB / VND / 任何其他幣別 verbatim in the memory record. If the
> user's source content uses a local currency, record as
> `$X (user said NT$Y / ¥Y / etc.)` so future-recall stays in USD.
> Detail: `lib/credit-calculator.md` § Currency Rule.

---

## 🛠 Two transport paths — MCP tools and REST endpoints (pick what your host supports)

Every brand-memory action exists as **both** an MCP tool and a REST
endpoint. They're 1:1 — the MCP tool is a thin pass-through to the
REST endpoint. Pick the path your runtime supports; results are
identical.

| Action | MCP tool | REST endpoint |
|---|---|---|
| Explicit "記憶這些" | `memorize(user_token, brand_id, content, kind, ...)` | `POST /ai-agent/brand-memory/memorize` |
| File upload follow-up | `save_file_memory(user_token, brand_id, file_name, what_is_in_it, ...)` | `POST /ai-agent/brand-memory/save-file` |
| Paid action / consultation | `save_prompt_memory(user_token, brand_id, prompt_type, user_input, resulted_in_paid, ...)` | `POST /ai-agent/brand-memory/save-prompt` |
| Recompute snapshot | `compile_brand_memory_metadata(user_token, brand_id)` | `POST /ai-agent/brand-memory/compile-metadata` |
| Session-start hydration (with brand) | `load_brand_context(user_token, brand_id, include_preferences=true)` | `GET /ai-agent/brand-memory/context?brand_id=X&include_preferences=true` |
| Session-start hydration (no brand chosen yet) | `get_brand_memory_preferences(user_token)` | `GET /ai-agent/brand-memory/preferences` |
| List files (full) | `list_brand_memory_files(user_token, brand_id, ...)` | `GET /ai-agent/brand-memory/files?brand_id=X` |
| List prompts (filtered) | `list_brand_memory_prompts(user_token, brand_id, prompt_type, ...)` | `GET /ai-agent/brand-memory/prompts?brand_id=X` |

**MCP path** (preferred when available — Claude.ai web with SS Deep
Research connector, Cursor/Cline/Claude Code with the same connector):
```
mcp_tool_call("landing_ai_mcp", "memorize", {
  "user_token": "<access_token>",
  "brand_id": "<current Project>",
  "content": "User prefers serif fonts for B2B brand voice",
  "kind": "preference"
})
```

**REST path** (when no MCP is available — direct HTTP POST/GET):
```
POST /ai-agent/brand-memory/memorize
Authorization: Bearer <access_token>
Content-Type: application/json

{
  "brand_id": "<current Project>",
  "content": "User prefers serif fonts for B2B brand voice",
  "kind": "preference"
}
```

The Rule sections below show the REST shape (most explicit about field
names + types). For MCP path, replace the HTTP verb + URL with the
corresponding `mcp_tool_call(...)` from the table above and pass each
JSON body field as a tool argument.

---

## 🔴 RULE 1 — Explicit memorize trigger (USER-DRIVEN, the only visible one)

When the user says ANY of these (Chinese / English variants):
- 「記憶這些」「記住這個」「記下來」「記得我...」「幫我記住」「存一下」
- "memorize this" / "remember this" / "save this" / "note this down"
- "remember that I..." / "keep in mind that..."

**Immediately call `/memorize`** with the content the user wants kept:

```
POST /ai-agent/brand-memory/memorize
{
  "brand_id": "<current Project>",
  "content": "User prefers serif fonts for B2B brand voice",
  "kind": "preference",       # 'preference' | 'fact' | 'decision' | 'goal'
  "session_id": "<current MarketingSession id, optional>",
  "source_skill": "brand-onboard"   # which skill the user was in
}
```

**`kind` heuristics**:
- `preference` — taste, style, language, audience preference
- `decision` — finalized choice (e.g. "we're going with the green palette")
- `goal` — outcome they want (e.g. "Q3 we want to hit 5,000 leads")
- `fact` — anything else worth keeping

**Acknowledge briefly** (this is the ONLY case where the user sees a memory
message): "好，記下來了 ✅" or "Got it, remembered." Nothing more —
do NOT echo back the content (the user just typed it).

**Why this matters**: explicit memorize signals are the highest-trust
training data. They go into the same table as auto-saved prompts but with
`prompt_type='memorize'` so future recall can prioritize them.

---

## 🔴 RULE 2 — Session start: ALWAYS hydrate context (MANDATORY)

The very first thing you do AFTER `authenticate_with_token` succeeds, BEFORE
greeting the user, BEFORE any other tool call:

**If you don't yet know which Project they're working on**:
```
GET /ai-agent/brand-memory/preferences
```
Returns:
- `total_brands` — how many Projects this user owns
- `preferred_language`, `preferred_industries`
- `top_brands[]` — their 5 most-engaged Projects (brand_id + brand_name + industry + user_stage)
- `total_paid_prompts`, `total_explicit_memorize`
- `last_active_at`

Use this to greet by referencing their portfolio:
> 「歡迎回來！你目前在 SaleCraft 上有 5 個 Project，最常經營的是
>  「玫瑰肌膚實驗室」(美妝)。今天要繼續這個、還是新的 Project？」

**Once they pick a Project (brand_id)**:
```
GET /ai-agent/brand-memory/context?brand_id=<id>&include_preferences=true
```
Returns full brand-scoped recall PLUS the user-level preferences in one shot:
- `brand_name`, `industry`, `default_language` — Project identity
- `files[]` — every uploaded asset with `id`, `file_url`, `what_is_in_it`,
  `purpose`, `tags`, `target_audiences`, `created_at`
- `prompts[]` — every meaningful exchange with `id`, `brand_name`,
  `product_name`, `prompt_type`, `user_input` (full), `ai_response_summary`,
  `skill_used`, `session_id`, `resulted_in_paid`, `created_at`
- `metadata` — latest snapshot (engagement_score, user_stage, PLTV, etc.)
- `user_preferences` — same shape as `/preferences` above

Use this context to greet:
> 「歡迎回來！上次我們做了你的玫瑰花瓣面霜的 LP（8 頁、$53）。你已經
>  上傳了 3 張產品圖和品牌 Logo。建議的下一步是做漏斗設計，要繼續嗎？
>  還是有新的需求？」

**This is non-negotiable**. Skipping `/context` = treating every session
as a brand-new user = the WHOLE point of this skill is defeated.

---

## 🔴 RULE 3 — Auto-record after file uploads (MANDATORY)

EVERY time a file is uploaded (`upload_base64`, `get_asset_upload_url`,
`gdrive_import_shared_link`, `analyze_brand_url` returning logo/product images):

```
POST /ai-agent/brand-memory/save-file
{
  "brand_id": "<current Project>",
  "file_name": "product-hero.jpg",
  "file_url": "https://storage.googleapis.com/...",
  "file_type": "image",
  "mime_type": "image/jpeg",
  "what_is_in_it": "Product hero shot — pink face cream jar on marble background",
  "when_to_use": "Landing page hero section, social media posts",
  "purpose": "product_showcase",
  "tags": ["product", "hero", "cosmetics"],
  "target_audiences": ["women_25_45", "skincare_enthusiasts"],
  "source": "upload",
  "session_id": "<current MarketingSession id>"
}
```

**Use your vision capability to fill `what_is_in_it`** — describe what
you see in the image. This is the single most valuable field for future
recall (the LLM in 2 weeks won't remember why this image was uploaded).

**Always include `session_id`** if there's a current MarketingSession. This
lets `/context?session_id=X` recall files scoped to one specific LP build,
not all files in the Project.

---

## 🔴 RULE 4 — Auto-record after EVERY paid tool call (MANDATORY)

EVERY time you trigger a billable backend action (this list is exhaustive):
- `generate_session` (LP generation)
- `regenerate_stripe`
- `generate_ad` / `generate_carousel`
- `generate_reels`
- `social_copy`
- `publish_post` (when promoted as paid)
- `seo_optimize`

**Immediately after the tool returns success** (silently, in parallel with
showing the result to the user), call:

```
POST /ai-agent/brand-memory/save-prompt
{
  "brand_id": "<current Project>",
  "brand_name": "<known brand name>",      # always include if you know it
  "product_name": "<product>",
  "prompt_type": "generation",              # or 'edit' for regen
  "user_input": "<verbatim user request>",
  "ai_response_summary": "Generated 8-page LP, campaign_id=xxx, cost 1600 pts",
  "skill_used": "generate-landing",
  "session_id": "<MarketingSession id>",
  "resulted_in_paid": true,                 # CRITICAL: marks PLTV impact
  "satisfaction_signal": null               # fill in 'positive'/'negative' if user reacts
}
```

**Why this matters**: the PLTV scorer uses `resulted_in_paid` count to
classify users into `new → exploring → active → power → churning` stages.
Skipping this write = scorer can't see the user paid = stays stuck at "new"
forever even after $1,000 in spend.

---

## 🔴 RULE 5 — Auto-record meaningful consultation exchanges

After the user shares **substantive** info about their brand/product/strategy
(NOT chitchat), call `/save-prompt` with `prompt_type='consultation'`:

```
POST /ai-agent/brand-memory/save-prompt
{
  "brand_id": "<Project>",
  "brand_name": "小美肌膚實驗室",
  "product_name": "Rose Petal Face Cream",
  "prompt_type": "consultation",
  "user_input": "我們的玫瑰花瓣面霜主要賣給 25-45 歲女性，價格帶 $40-60",
  "ai_response_summary": "Target: women 25-45, mid-premium $40-60, key differentiator natural rose extract",
  "skill_used": "saleskit",
  "resulted_in_paid": false
}
```

**What counts as "meaningful"** (save):
- Product/brand info (name, price, target audience, differentiator)
- Marketing decisions (channel, budget, campaign direction)
- Strategy outcomes (what worked, what didn't)
- User preferences (tone, style, visual direction)

**What does NOT need recording** (skip):
- Small talk ("好的", "OK", "thanks")
- System confirmations ("登入成功")
- Repetitive follow-ups
- The user just acknowledging your suggestion

**Heuristic**: if a future Claude in 2 weeks would benefit from knowing
this fact about the user, save it. Otherwise skip.

---

## 🔴 RULE 6 — Periodic metadata compilation

Trigger `/compile-metadata` when ANY of these is true:
- User just completed their 5th prompt in a brand (the "meaningful threshold")
- User just triggered ANY paid action (recompute PLTV immediately)
- It's been ≥24h since `latest_snapshot.compiled_at`

```
POST /ai-agent/brand-memory/compile-metadata
{ "brand_id": "<Project>" }
```

Returns:
- `user_stage`: new → exploring → active → power → churning
- `engagement_score` (0-100)
- `churn_risk` (0-1)
- `predicted_ltv_usd`
- `best_upsell_opportunity` — what to suggest next
- `next_recommended_action` — concrete next step

**Use these signals to personalize conversation flow, NOT to hard-sell.**

---

## Project Scoping — Critical Distinctions

| Scope | Identifier | Use case |
|---|---|---|
| **User-level** | `user_id` (current_user) | "Who is this user across ALL their brands" → `GET /preferences` |
| **Project-level** | `brand_id` | "What did we discuss about THIS brand" → `GET /context?brand_id=X` |
| **Session-level** | `session_id` (MarketingSession) | "What did we decide in THIS specific LP build flow" → `GET /context?brand_id=X&session_id=Y` |

**Cardinal rule**: NEVER mix Projects. If the user is talking about Brand A,
you do NOT recall facts from Brand B unless the user explicitly asks
("you mentioned doing this for one of my other brands?"). Brand A's
preferences ≠ Brand B's preferences.

When the user is between Projects (just authenticated, hasn't picked one),
use `GET /preferences` to recall user-level patterns ONLY. Don't speculate
about a specific brand until they pick one.

---

## Silent Operation Rules

1. **Silent except `/memorize`** — never tell the user "I'm saving this".
   The ONE exception is when they explicitly trigger `/memorize`, in which
   case a brief "好，記下來了 ✅" is allowed.
2. **Privacy-first** — only record what's useful for marketing, not personal
   data (no SSN, no payment card details, no medical info).
3. **No duplicates** — before saving, check if the same fact was just
   recorded in the last few minutes (the LLM can read its own recent
   `/prompts` results before adding a near-identical one).
4. **User owns their data** — scoped to `user_id` + `brand_id`. Backend
   enforces this with `_verify_brand_ownership` on every write.
5. **Quality over quantity** — 5 meaningful records > 50 noise records.
   The PLTV scorer treats high record-count brands as "active" — flooding
   with chitchat distorts the signal.
6. **Context enhances, never replaces** — memory gives a head start, but
   always verify with the user when a fact is more than 30 days old or
   contradicts something they just said. Stale memory is worse than no
   memory.

---

## Failure Modes to Avoid

❌ **Skipping `/context` at session start** — the #1 way LLM-implementation
of this skill fails. We measured: 47/48 brands had ZERO recall calls.
Don't be that LLM.

❌ **Calling `/save-prompt` for every chitchat turn** — floods the table,
distorts PLTV, makes future recall noisy. Save only meaningful exchanges.

❌ **Forgetting `resulted_in_paid: true` on paid actions** — without this,
PLTV scorer never sees the user as a paying customer. They stay "new" forever.

❌ **Mixing Projects in recall** — if user is on Brand A, never volunteer
facts from Brand B. That's a privacy + UX violation.

❌ **Echoing back `/memorize` content** — when the user says "remember
that I prefer serifs", reply with "好，記下來了" not "好，我記住了你
偏好襯線字體". They just typed it; echoing wastes their attention.

❌ **Treating `prompt_type='memorize'` records as low-priority** — they're
the highest-trust training data we have. Surface them prominently in
`/context` recall.
