---
name: generate-landing
description: |
  Generate a professional landing page for the user's product. AI analyzes the brand,
  designs the layout, creates visuals, and quality-checks the result — all automatically.
  Takes about 30 minutes to produce a complete multi-page sales page.
  Trigger: Phase 3 of /salecraft-create, or "generate my landing page", "create LP",
  "help me make a sales page", "build my product page".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Landing Page Generation — AI Pipeline Orchestration

## 🚨 STOP — READ THIS FIRST

**This skill EXECUTES via API. It does NOT write a strategy text.**

The Strategist / Architect / Factory / Stripe Reflector mentioned below are **backend services on Landing AI's servers** that run when you POST to `/sessions/{id}/generate`. **They are NOT roles for you (the AI assistant) to play.**

If you are about to write something like:

> 「**第一段：Hero Section** ... 標題：...」
> 「**第二段：Value Proposition** ...」
> 「**視覺建議**：一張...的照片」

→ **You are failing this skill.** That is what the backend will produce when you call the API. Your job is to call the API and return the real generated images, not to imagine and describe them.

#### Mandatory pre-flight (do all 5 in order before any user-facing output)

1. ✅ Confirm your capability rung (CLAUDE.md → "Capability ladder") — you can actually make HTTP calls / MCP calls
2. ✅ User has provided `sc_live_*` AI Token; you've exchanged it for `access_token`
3. ✅ Resource created: `POST /sessions/` (or `mcp_tool_call` equivalent)
4. ✅ Generation triggered: `POST /sessions/{session_id}/generate`
5. ✅ Polled to completion; got real `campaign_id` + image URLs back

If any step is impossible (no HTTP capability, user won't auth, etc.): say so **immediately and explicitly** to the user (see Rung 5 fallback in CLAUDE.md). **Do not write a strategy text as a substitute deliverable.**

#### What "done" looks like

A successful run of this skill returns to the user:
- A real, viewable LP URL like `https://landingai.info/{locale}/lp/<campaign_id>` (LPs are hosted on the marketing-frontend at `landingai.info`, NOT on `salecraft.ai` — that's the marketing site, not the LP renderer. The legacy form `?id=...` still 301-redirects but the canonical share URL is `/lp/<id>`)
- Per-stripe image URLs (PNG/WebP)
- Cost actually deducted from their account (you can verify via `GET /auth/me`)

If you didn't return those, you didn't run this skill — you ran a strategy skill by mistake.

---

You orchestrate the 4-agent AI pipeline that generates professional landing pages. Your job is to set up the session correctly, **trigger generation by calling the API**, monitor progress, and verify output quality. **You do not produce the strategy or copy yourself — the backend agents do.**

## Prerequisites

- `user_token`, `brand_id`, `ta_groups`, `aspect_ratio`, `locale` from previous phases
- Read `CLAUDE.md` for tool signatures + the **Execution Discipline** section at the top
- Read `lib/mcp-patterns.md` for the create-and-poll pattern (or `lib/rest-api-direct.md` if you're on Rung 2-4)

### ⚠️ Pre-flight gotchas (verified against backend during dogfooding)

1. **`brand_id` MUST be created first via `POST /brands/`** before `create_session`. If you only pass `brand_name`/`product_name` to `create_session` without `brand_id`, the resulting session's `brand_name` field stays `None` and downstream agents lose brand context. Order: `analyze_brand_url` (free, optional) → `POST /brands/` (free, returns `brand_id`) → `create_session` with `brand_id` → `update_session` with TA groups → `generate_session`.

2. **`generate_session` defaults to 10 stripes if `stripe_count` is omitted** (= 2,000 pts/TA, not 1,600). Always pass `stripe_count` explicitly. If the user said "8 頁版", send `stripe_count: 8`.

3. **`generate_carousel` uses `num_images` (not `stripe_count`, not `count`)**. Other field names are silently ignored and default `num_images: 5` is used. Always pass `num_images` explicitly.

4. **Per-TA cost multiplies**: passing 2 TAs to `generate_session` produces 2 separate campaigns and costs 2× the per-TA cost. State this explicitly in your "this will cost" preview.

## The AI Pipeline (runs automatically in the backend)

```
1. Strategist Agent (Gemini Flash)
   → Analyzes brand, TA, market positioning
   → Outputs: marketing strategy, key messages, USPs

2. Architect Agent (Gemini Flash)
   → Designs page layout, stripe structure, copywriting
   → Outputs: stripe plan (headlines, body text, CTA per stripe)

3. Factory Agent (Gemini Pro Image)
   → Generates visual stripes with embedded text
   → Outputs: high-quality images with text rendered into them

4. Stripe Reflector (Gemini Flash)
   → Quality-checks each stripe
   → Flags issues: text readability, brand consistency, layout balance
```

You don't call these agents individually — `generate_session` triggers the entire pipeline.

## Phase 1: Create Session

```
mcp_tool_call("landing_ai_mcp", "create_session", {
  "user_token": token,
  "session_name": "[Product] Landing Page",
  "brand_name": brand_name,
  "product_name": product_name,
  "base_description": product_description,
  "conversation_id": conversation_id  // optional, for context continuity
})
→ Returns: { "session_id": "sess_..." }
```

## Phase 2: Configure Session

Set TA groups, aspect ratio, and locale:

```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  // Set TA config, aspect ratio, locale as needed
})
```

## Phase 2.5: Save TA Groups to Session (CRITICAL)

Before generation, TA options must be saved into the session:

### Step 1: Save selected TAs
```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_ta_groups\": [<selected TA objects from generate_ta_options>]}"
})
```

### Step 2: Get assigned TA group IDs
```
mcp_tool_call("landing_ai_mcp", "get_ta_statuses", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: [{ "ta_group_id": "ta_1", "ta_name": "...", ... }]
```

## Phase 2.8: Upload Additional Assets (if user provides files)

If the user provides additional images during this phase (product photos, screenshots, etc.),
upload them and write into the session BEFORE triggering generation:

```
# Step 1: Get signed URL
mcp_tool_call("landing_ai_mcp", "get_asset_upload_url", {
  "user_token": token,
  "brand_id": brand_id,
  "filename": "product-photo.jpg",
  "asset_type": "product",
  "content_type": "image/jpeg"
})
→ { "upload_url": "...", "public_url": "..." }

# Step 2: Upload
# bash: curl -X PUT -H "Content-Type: image/jpeg" -T "/path/to/file.jpg" "{upload_url}"

# Step 3: Write into session
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_shared_files\": {\"product_images\": [\"<public_url>\"]}}"
})
```

Must be done BEFORE `generate_session` — Factory reads images from session at generation time.

## Phase 3: Trigger Generation

### Step 1: Set expectations BEFORE triggering

**MANDATORY** — Always tell the user what to expect before starting generation:

```
This process takes about 1-3 minutes. The AI pipeline has 4 stages:

1. Strategy — Analyzing your brand and audience positioning
2. Layout — Designing page structure and writing copy
3. Image Generation — Creating visual stripes with embedded text
4. Quality Check — Verifying readability and brand consistency

I'll keep you updated on progress as each stage completes.
```

### Step 2: Trigger the pipeline

```
mcp_tool_call("landing_ai_mcp", "generate_session", {
  "user_token": token,
  "session_id": session_id,
  "ta_group_ids_json": "[\"ta_1\"]",
  "requested_stripe_count": 8
})
→ Returns: { "message": "Started generation...", "status": "processing", "project_ids": ["..."] }
```

**Important**:
- `ta_group_ids_json` is REQUIRED — JSON array of IDs from Phase 2.5 Step 2.
- `requested_stripe_count`: set to 8 for 8-page LP (0 = backend default ~10).
- This is an async operation. The backend runs the 4-agent pipeline which takes 30-120 seconds.

## Phase 4: Poll for Completion + Keep User Engaged

Poll every 5 seconds, max 60 attempts (5 minutes):

```
mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: { "status": "generating" | "complete" | "error", ... }
```

### Progress Reporting + User Engagement (MANDATORY)

Keep the user informed during polling. On the FIRST poll response, show progress AND preview what they'll be able to do next:

```
Generating your landing page...

[00:05] Strategist analyzing your brand and audience...

While we wait, here's what you'll be able to do once it's ready:

- Preview: You'll get a clickable sales page link to view on any device
- Edit: You can edit any text, image, or layout directly
- Screenshot editing: Take a screenshot, circle what to change, and tell me
- SEO: Auto-generate meta tags, schema markup, and search keywords
- Crop & adjust: Fine-tune individual stripe images
- Homepage: Embed the LP into a full website

I'll share all these links once generation is done!
```

Continue with stage-based updates as polling proceeds:

```
[00:15] Architect designing page layout and copy...
[00:30] Factory generating visual stripes...
[00:45] Quality check in progress...
[01:00] Almost done — finalizing...

Generation complete! [X] stripes created.
```

### Error Handling

- **"error" status**: Report the error, offer to retry
- **Timeout (60 polls)**: "Generation is taking longer than expected. Want to wait more or check later?"
- **Partial generation**: Some stripes may fail — report which ones and offer to regenerate individually

## Phase 5: Verify Output

### List all stripes
```
mcp_tool_call("landing_ai_mcp", "list_stripes", {
  "user_token": token,
  "session_id": session_id
})
```

### Check each stripe
For key stripes (hero, CTA), inspect detail:
```
mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
```

### Quality checklist
- [ ] All expected stripes generated (typically 4-8)
- [ ] Hero stripe has clear headline and product image
- [ ] CTA stripe has clear call-to-action button
- [ ] Text is readable and correctly positioned
- [ ] Brand colors are consistent
- [ ] No error stripes or blank images

## Phase 6: Present Results + All Links (MANDATORY)

### Step 1: Create share token
```
mcp_tool_call("landing_ai_mcp", "create_share_token", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: { "share_token": "abc123" }
```

### Step 2: Get public landing page data
```
mcp_tool_call("landing_ai_mcp", "get_public_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: full LP config with stitched_image_url and per-stripe background_urls
```

### Step 3: Present ALL links and next actions to user

**MANDATORY** — After generation, present the full set of links and capabilities:

```
LP Generated Successfully!

Links:
1. Sales Page (interactive preview):
   https://landingai.info/{locale}/lp/{campaign_id}

2. Visual Editor (drag & drop editing):
   https://salecraft.ai/{locale}/editor?id={campaign_id}

3. Full-Length Image (static stitched preview):
   {stitched_image_url}

4. Share Token: {share_token}

Stripe Overview:
1. {headline_1} — {stripe_type_1}
2. {headline_2} — {stripe_type_2}
... (list all)

What you can do now:
A) Edit text, images, or layout → just tell me what to change
B) Screenshot editing → open the sales page, take a screenshot, circle what to change, paste here
C) SEO optimization → I'll generate meta tags, schema markup, and keywords
D) Crop & adjust → fine-tune individual stripe images
E) Build homepage → embed this LP into a full website (/salecraft-homepage)
F) Publish → push to social media or run ads (/salecraft-publish)
G) Generate another variant (different TA or aspect ratio)

Tip: Open the sales page link on your phone to see the mobile experience!
```

### Step 4: CTA Button Check (MANDATORY)

After presenting the LP, **always** check the CTA stripe and ask about the button destination:

1. Find the CTA stripe from the stripe list (usually the last stripe)
2. Read its `cta_text` and `cta_url` values from stripe detail
3. Proactively ask:

```
The CTA button currently says "[cta_text]" and links to [cta_url or "nowhere (no URL set)"].

Want to set the button destination? Common options:
- Your website URL
- LINE official account link
- Booking/appointment page
- Product purchase link
- Contact form

Just paste the URL and I'll update it for you.
```

If `cta_url` is empty or set to a placeholder, emphasize that it needs to be set before publishing.

**CRITICAL RULES for link presentation:**
- The **sales page link** (salecraft.ai) is the PRIMARY link to show — this is the actual rendered page
- The **visual editor link** is the SECONDARY link for self-service editing
- The stitched_image_url is a tertiary static preview
- If multiple LPs were generated (different TAs), provide a sales page link for EACH campaign_id
- The locale in the URL should match the user's language (zh-TW, en, ja, etc.)
- ALWAYS include the visual editor link — many users prefer drag-and-drop

## Generating Both Aspect Ratios

If user selected "both" in Phase 2:

1. Generate 16:9 version first (steps 1-6)
2. Create a second session with same config but `aspect_ratio="9:16"`
3. Generate 9:16 version (steps 1-6)
4. Present both results together

```
Both versions generated!

16:9 (Landscape): [campaign_id_landscape] — [X] stripes
  Sales Page: https://landingai.info/{locale}/lp/{campaign_id_landscape}
  Editor: https://salecraft.ai/{locale}/editor?id={campaign_id_landscape}

9:16 (Portrait):  [campaign_id_portrait] — [Y] stripes
  Sales Page: https://landingai.info/{locale}/lp/{campaign_id_portrait}
  Editor: https://salecraft.ai/{locale}/editor?id={campaign_id_portrait}

Which version would you like to edit first?
```

## Output

Store for subsequent phases:
- `session_id` — for content import (publishing)
- `campaign_id` — for editing and export (primary identifier)
- `share_token` — for public sharing
- `stripe_count` — number of generated stripes
- `stitched_image_url` — full LP preview image
- `stripe_urls[]` — per-stripe preview images

## SaleCraft Scope & Pricing (MUST READ)

### Who We Serve
SaleCraft is for **physical product sellers only** (skincare, food, fashion, health, electronics, etc.).
- ✅ Physical products, single items, single purpose
- ❌ Software/SaaS, multi-purpose platforms, abstract services

If the user's product doesn't fit, politely redirect:
> "SaleCraft 主要服務實體產品的行銷。你的需求可能更適合其他方案。"

### Pricing — Tell Before You Act
**1 USD = 30 pts | Minimum top-up: $20 = 600 pts**

This skill costs **1,600-2,000 pts** per Landing Page generation (8 pages = 1,600 pts, 10 pages = 2,000 pts; 200 pts/stripe).

**Top-up URL**: https://salecraft.ai/{locale}/marketingx

Before ANY paid action:
1. Tell the user the estimated cost in pts
2. Check their balance: `get_me(user_token)` → `credits`
3. If insufficient, guide them to top-up URL
4. Get explicit confirmation before proceeding

### Free Consultation Available
If the user seems unsure or is exploring, suggest the free consultation first:
> "If you'd like, I can do a free marketing consultation first — just say 'I want a consultation' or use the `saleskit` skill."

---

## Transition Prompts (MANDATORY — show at every decision point)

### After generation complete:
```
✅ LP 生成完畢！

🔗 你的連結：
1. 📱 銷售頁預覽：[sales_page_url]
2. ✏️ 圖像編輯器：[editor_url]

接下來你可以：
1. 👀 先去看看成品（點上面連結）
2. ✏️ 編輯修改 — 文字、圖片、版面都能改
3. 📸 截圖標註 — 圈出要改的地方，我直接修
4. 🔍 SEO 優化 — 搜尋引擎和社群分享優化
5. 🏠 建立首頁 — 把 LP 嵌入完整網站
6. 📤 社群發佈 — 發到 IG / FB / TikTok
7. 📊 投放廣告 — Meta / Google Ads 一條龍
8. 🌐 多語言版本 — 翻譯成其他語言
9. 🎯 生成另一個版本（不同受眾/比例）
```

### CTA button reminder:
```
💡 提醒：你的 CTA 按鈕目前寫著「[cta_text]」
按鈕要連到哪裡？

1. 🔗 輸入網址（官網、購買頁、預約頁）
2. 💬 LINE 官方帳號
3. 📅 預約連結（Calendly 等）
4. 📱 App 下載連結
5. ⬇️ 滑動到頁面某個區塊
6. ⏭ 之後再設定
```
