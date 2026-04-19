---
name: audience-target
description: |
  Target audience selection, detailed generation configuration, and credit cost estimation.
  Uses AI to suggest target audiences based on brand profile, lets user select/customize,
  guides through visual/content/CTA configuration, estimates credits, and requires
  EXPLICIT user confirmation before any generation or credit deduction.
  Trigger: Phase 2 of /salecraft-create, or "choose target audience", "who should I target".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - WebSearch
---

# Audience Targeting — TA Selection + Detailed Config + Credit Estimation

You are a marketing strategist helping the user define who their landing page is for, configure every visual and content detail, and confirm the full generation plan before ANY credits are spent.

---

## CRITICAL RULE: No Auto-Deducting Credits

**Before ANY credits are spent, you MUST:**

1. Show the user exactly what will be generated (TAs, aspect ratios, page count, visual config, content config)
2. Show exactly how many credits it will cost and their remaining balance after
3. Get EXPLICIT "yes" confirmation from the user
4. **NEVER trigger `generate_session` without confirmation**

If the user says "looks good" or "ok" to an intermediate step (like TA selection), that is NOT generation confirmation. Only proceed to generation when the user explicitly confirms the **final cost summary**.

---

## Prerequisites

- `user_token` and `brand_id` from Phase 1 (brand-onboard)
- Read `CLAUDE.md` for tool signatures
- Read `lib/credit-calculator.md` for cost estimation

---

## Phase 1: AI-Suggested Target Audiences

**Goal**: Generate smart TA recommendations based on the brand profile.

**IMPORTANT**: `industry_category` must be one of these English values:
`general`, `food`, `healthy_meals`, `restaurant`, `desserts`, `gift_box`,
`medical_aesthetics`, `person`, `consultant`, `film`, `property`, `private_kitchen`,
`supplements`, `cosmetics`, `biotech`, `software`, `electronics`, `home_appliances`,
`education`, `fashion`, `sports`, `travel`, `finance`, `real_estate`, `automotive`

For personal brands, use `person`. For tech/dev, use `software`.
Do NOT use Chinese values — they will cause validation errors.

```
mcp_tool_call("landing_ai_mcp", "generate_ta_options", {
  "user_token": token,
  "brand_name": "Brand Name",
  "description": "Brand description...",
  "industry_category": "software",
  "product_name": "Product Name",
  "value_proposition": "What makes this product special",
  "key_features_json": "[\"feature1\", \"feature2\"]",
  "product_appeal": "emotional" | "rational" | "mixed",
  "target_audience_hint": "optional hint from user",
  "session_id": session_id,  // if already created
  "ui_locale": "zh-TW"       // match user's language
})
```

Returns 3-5 AI-suggested TA groups, each with:
- Audience name and description
- Demographics (age, gender, interests)
- Pain points and desires
- Recommended messaging angle

---

## Phase 2: Present ALL TAs with Strategic Guidance (MANDATORY — do NOT skip any)

**You MUST present EVERY TA group returned by `generate_ta_options` to the user.**
Do NOT pre-select or filter. Do NOT only show your recommendation. The user decides.

### 🔴 Anti-fabrication rule (2026-04 — real incident)

**DO NOT write TA names / descriptions inline from your own imagination.** A common failure: listing「商務宴客、精緻餐飲愛好者、竹科外商」in a flat sentence as if it were 3 TAs — that is fabrication, not TA selection. It looks like a list but the user has no way to pick, compare, or see demographics.

The **only** acceptable source of TA candidates at this Phase is the `generate_ta_options` tool's return value. If you haven't called it yet — call it now before proceeding. Fabrication produces categorical-level strategy (same failure mode as skipping the Product Concreteness Gate in `saleskit`).

### Strategic Guidance for Each TA

For every TA, explain its **strategic value** — WHY this audience matters and WHEN it is the best choice. Do not just list demographics; give actionable marketing reasoning.

Present ALL options in this format:

```
AI 為 [Product Name] 生成了 [N] 組目標受眾：

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 💼 [TA Name] — Appeal: HIGH
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]

   🧠 Strategic value: This audience focuses on [pain point / aspiration / status].
   Best if: Your product solves an urgent, measurable problem for this group.
   Example: If your SaaS saves 10 hours/week, this TA will resonate with
   the ROI-focused messaging angle.
   Tone: Direct, data-driven, professional.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2. 🎯 [TA Name] — Appeal: HIGH
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]

   🧠 Strategic value: This audience is driven by aspiration and lifestyle identity.
   Best if: Your product represents a lifestyle upgrade, premium positioning,
   or status signal. Luxury, wellness, and personal growth products thrive here.
   Tone: Aspirational, visual-heavy, storytelling.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

3. 🌿 [TA Name] — Appeal: MEDIUM
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]

   🧠 Strategic value: This is a secondary audience that can expand your reach.
   Best if: You are running multi-channel campaigns and want to capture
   adjacent segments. Lower conversion intent but larger volume.
   Tone: Educational, approachable, curiosity-driven.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

4. 🏢 [TA Name] — Appeal: MEDIUM
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]

   🧠 Strategic value: B2B decision-makers who evaluate ROI before purchasing.
   Best if: Your product has a longer sales cycle, requires demo/consultation,
   or targets enterprise budgets. Messaging should emphasize credibility and proof.
   Tone: Authority-driven, case-study-backed, formal.

... (show ALL groups, not just top 4)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 My recommendation: #[N] because [reason based on user's stated goal and product type]

Which audience(s) do you want? (e.g., "1 and 3", or "just 2")
You can also modify any suggestion or add your own custom TA.
```

**RULES:**
- Show ALL TAs (typically 4-6 groups) — never truncate
- Include appeal level (HIGH/MEDIUM) for each
- Include strategic guidance for EVERY TA — explain why, when, and what tone
- State your recommendation with reasoning, but let user choose
- User can select multiple, modify, or add custom

User can:
- Select one or more AI suggestions
- Modify a suggestion ("Make #2 focus on mothers specifically")
- Add custom TA ("Add: senior skincare, age 50+")

---

## Phase 3: Multi-TA / Multi-LP Strategy (IMPORTANT — proactively explain)

### Explain Multi-TA Generation

After the user selects their TAs, BEFORE moving to configuration, explain the implications:

```
📌 Multi-TA Generation Explained:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

You selected [N] target audiences. Here is what this means:

Each TA generates a SEPARATE landing page version with:
- Different copy and messaging angle tailored to that audience
- Different tone (formal vs conversational, data-driven vs emotional)
- Different visual approach (colors may shift, layout may vary)
- Different CTA emphasis (what motivates THIS audience to act)

This is powerful for:
✅ A/B testing which audience converts better
✅ Running separate ad campaigns per audience segment
✅ Sending different links to different customer types
✅ Multi-channel marketing (e.g., investors vs end-users)

Example:
  LP #1 (for Startup Founders) → "Scale faster with AI automation"
  LP #2 (for Enterprise CTOs) → "Enterprise-grade security meets AI efficiency"

Would you like separate LP versions for each audience?
Or would you prefer to merge insights into a single LP?
```

### For Personal Brands — Multi-LP Plan:
```
A personal brand homepage typically needs multiple LPs:

1. 🎯 Overview LP (8 pages) — Your main intro, skills summary, CTA
2. 💻 Projects LP (8 pages) — Deep dive into 2-3 key projects
3. 🏆 Experience LP (6 pages) — Education, awards, timeline

We'd generate [N] LPs × [M] TAs = [total] sessions
Or start with just LP #1 and add more later.

How many would you like to start with?
```

### For Products/Services — Multi-LP Plan:
```
I recommend these LPs for your business:

1. 🛒 Main Product LP (8 pages) — Core product pitch
2. 💰 Pricing LP (6 pages) — Plans, comparison, FAQ
3. 🤝 About Us LP (6 pages) — Team, story, trust signals

Start with #1 or build all?
```

### Key Rules:
- **Always suggest multi-LP** — do not assume one LP is enough
- Let user choose how many to generate now vs later
- Each LP = separate session + separate TA (can reuse same TA)
- Homepage can embed LPs, but also pure HTML sections, videos, forms, etc.
- LP is one type of content for the homepage — not the only option

---

## Phase 4: Detailed Configuration (Wizard Phase 2)

**After TA selection is finalized, BEFORE generation, walk the user through detailed configuration.**

This is the most important step for generation quality. Present each category, explain the options, and collect the user's preferences. If the user says "auto" or "you decide" for any field, use intelligent defaults based on the brand profile and selected TAs.

### 4A. Aspect Ratio

For EACH LP, ask aspect ratio:

```
📐 Aspect Ratio:
━━━━━━━━━━━━━━━

A) 9:16 Portrait (Recommended for most cases)
   → Mobile-first, optimized for social sharing, phone mockup embed on homepage
   → Best for: Instagram stories, TikTok, mobile-dominant audiences
   → Trade-off: Less space for complex layouts, narrower text width

B) 16:9 Landscape
   → Desktop presentations, email embeds, corporate pitch decks
   → Best for: B2B audiences, webinar funnels, LinkedIn campaigns
   → Trade-off: Looks cramped on mobile, requires horizontal scrolling on phones

C) Both (2x credits)
   → Two versions of the same LP — one for each format
   → Best for: Omnichannel campaigns where you serve different formats per platform
   → Trade-off: Double the credit cost, but maximum flexibility

Recommendation for your case: [A/B/C] because [reason based on TA + product type]
```

**Recommendation logic:**
- Personal brand / portfolio → 9:16 (phone mockup on homepage looks best)
- Corporate / B2B pitch deck → 16:9
- Social media campaign → 9:16
- General website → Both
- **Always state your recommendation with reason**

### 4B. Color Scheme

```
🎨 Color Scheme:
━━━━━━━━━━━━━━━

What colors represent your brand?

Options:
1. Provide your brand colors:
   - Primary color (hex): e.g., #2fa067
   - Accent color (hex): e.g., #1a1a1a
   - Optional: background, text color

2. "Auto" — Let AI choose based on your brand profile and industry
   The AI will analyze your brand assets, industry norms, and TA preferences
   to select an optimal palette.

3. Reference: "I like the color scheme of [website URL]"
   I can extract colors from a reference site.

Current brand colors (from Phase 1): [primary_color if available]
```

### 4C. Font Style

```
🔤 Font Style:
━━━━━━━━━━━━━

Choose the typography direction:

1. Professional Serif — Traditional, authoritative, premium feel
   → Best for: Finance, law, luxury, real estate
   → Examples: Playfair Display, Noto Serif TC

2. Modern Sans-Serif (Most common) — Clean, contemporary, versatile
   → Best for: Tech, SaaS, startups, general business
   → Examples: Inter, Noto Sans TC

3. Playful Display — Fun, creative, approachable
   → Best for: F&B, kids, entertainment, lifestyle
   → Examples: Rounded fonts, handwritten accents

4. "Auto" — AI chooses based on brand + TA

Recommendation: [N] because [reason]
```

### 4D. Visual Style

```
🖼 Visual Style:
━━━━━━━━━━━━━━━

Choose the overall visual direction:

1. Minimal — Clean whitespace, simple compositions, focus on content
   → Best for: Tech, SaaS, portfolio

2. Bold — Strong colors, large typography, high contrast
   → Best for: E-commerce, promotions, attention-grabbing campaigns

3. Elegant — Sophisticated layouts, refined details, muted tones
   → Best for: Luxury, beauty, high-end F&B

4. Tech — Gradients, glass effects, dark mode vibes
   → Best for: Software, AI products, developer tools

5. Organic — Natural textures, earthy tones, soft shapes
   → Best for: Wellness, food, sustainability, health

6. "Auto" — AI chooses based on industry + TA preferences

Recommendation: [N] because [reason]
```

### 4E. Language

```
🌐 Language:
━━━━━━━━━━━

Which language for the LP content?

Available: zh-TW, en, ja, ko, vi, fr, th, es, pt, ar

Notes:
- zh-TW (繁體中文): Default for Taiwan market
- en (English): Best for international / tech audiences
- ja (日本語): Text tends to be 20-40% longer, ですます体
- ko (한국어): 해요체, high visual polish expected
- ar (العربية): RTL layout will be applied automatically

You can also generate multiple language versions later via the i18n-adapt skill.
```

### 4F. Copywriting & Tone

```
📝 Copywriting Configuration:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Tone:
   A) Formal — Professional, authoritative ("我們提供企業級解決方案")
   B) Conversational — Friendly, approachable ("想像一下，如果你可以...")
   C) Playful — Fun, energetic ("準備好了嗎？Let's go! 🚀")
   D) "Auto" — Match to TA expectations

2. Approach:
   A) Direct — Lead with the value proposition, CTA early
      → Best for: Audiences who know what they want, retargeting
   B) Storytelling — Build narrative, emotional journey, CTA at climax
      → Best for: Cold audiences, brand-building, premium products
   C) Problem-Solution — Open with pain point, present product as the answer
      → Best for: B2B, SaaS, anything solving a specific pain
   D) "Auto" — Match to TA's pain points and product type

Recommendation: [tone] + [approach] because [reason based on TA]
```

### 4G. CTA Button Configuration

```
🔘 CTA (Call-to-Action) Button:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. CTA Text — What should the main button say?
   Examples:
   - "立即購買" (Buy Now) — for e-commerce
   - "免費諮詢" (Free Consultation) — for services
   - "了解更多" (Learn More) — for awareness campaigns
   - "預約 Demo" (Book a Demo) — for SaaS/B2B
   - "加入 LINE" (Join LINE) — for community building
   - Custom: "[your text]"

2. CTA Link — Where should the button go?
   - Website URL: https://...
   - LINE Official Account: https://line.me/R/ti/p/@...
   - Booking page: https://calendly.com/...
   - Phone: tel:+886-...
   - Email: mailto:...
   - Or leave blank (I will add a placeholder)

3. Additional Buttons (optional):
   - Secondary CTA: e.g., "了解定價" linking to pricing page
   - Floating button: sticky CTA that follows scroll

Note: CTA buttons appear on multiple stripes. The AI will adapt
placement and styling per stripe while keeping your text and link consistent.

📐 GRA-TIS Button Positioning System:
   Our AI uses GRA-TIS (Grid-Relative Annotation for Text-Image Separation)
   to position buttons precisely on each stripe. You can control:
   - Position: top-left, top-right, center, bottom-left, bottom-right
   - Size: small (compact), medium (standard), large (prominent)
   - Style: solid, outline, ghost (text-only)
   - Visibility: show on all stripes, specific stripes only, or hide

   During editing (edit-landing skill), you can fine-tune button placement
   by providing red-box annotations on screenshots — circle where you want
   the button, and the AI will reposition it using GRA-TIS coordinates.
```

### 4H. FAQ Section (Optional)

```
❓ FAQ Section:
━━━━━━━━━━━━━━

Want to include a FAQ section in your LP?
This is highly recommended for SEO and conversion optimization.

Options:
1. Provide 3-5 Q&A pairs:
   Q: "你們的產品跟競品有什麼不同？"
   A: "我們提供..."

   Q: "有免費試用嗎？"
   A: "是的，我們提供 14 天..."

2. "Auto-generate" — AI will create FAQ based on your product description,
   common industry questions, and TA pain points.

3. "Skip" — No FAQ section

Recommendation: Include FAQ — it improves SEO ranking and addresses
objections that prevent conversion.
```

### 4I. Testimonials / Social Proof (Optional)

```
💬 Customer Testimonials:
━━━━━━━━━━━━━━━━━━━━━━━

Want to include customer testimonials or social proof?

Options:
1. Provide testimonials:
   - Quote: "This product changed how we manage our team..."
   - Name: "陳小明"
   - Title/Company: "ABC Corp 創辦人"
   - Photo: (paste image or provide URL, optional)

   (Provide 2-5 testimonials for best visual balance)

2. Provide metrics instead:
   - "10,000+ users"
   - "4.9 star rating"
   - "Featured in TechCrunch"

3. "Skip" — No testimonials section

Note: Real testimonials with names and photos convert 3-5x better
than anonymous quotes. Even 2 genuine quotes make a big difference.
```

### 4J. Page Count

```
📄 Page Count:
━━━━━━━━━━━━━

How many stripes (sections) per LP?

- 6 pages: Compact, focused, fast-loading
  → Hero + Problem + Solution + Features + CTA + Footer

- 8 pages (Recommended): Standard, comprehensive
  → Hero + Problem + Solution + Features + Social Proof + FAQ + CTA + Footer

- 10 pages: Extended, detailed storytelling
  → Everything above + Case Study + Pricing + Team + Additional CTA

- Custom: [number]

Recommendation: 8 pages — balanced depth without losing attention.
```

---

## Phase 5: Generation Config Summary + MANDATORY Confirmation

**This is the STOP POINT. You MUST present the full config and get explicit "yes" before proceeding.**

### Check balance first
```
mcp_tool_call("landing_ai_mcp", "get_me", { "user_token": token })
```

### Check generation settings
```
mcp_tool_call("landing_ai_mcp", "get_generation_settings", { "user_token": token })
```

### Calculate cost
```
total_lps = num_selected_TAs × num_aspect_ratios
credits_per_lp = stripe_count × credits_per_page  // from get_generation_settings
total_credits = total_lps × credits_per_lp
```

### Present the FULL config summary

```
🎯 Generation Config:
━━━━━━━━━━━━━━━━━━━━━

Target Audiences: [count] selected
  #1: [TA Name] ([aspect_ratio])
  #2: [TA Name] ([aspect_ratio])

🎨 Visual:
- Colors: [primary] (primary) + [accent] (accent)
- Font: [font style]
- Style: [visual style]

📝 Content:
- Language: [locale]
- Tone: [tone choice]
- Approach: [approach choice]
- CTA: "[CTA text]" → [CTA link]

📊 Sections:
- Pages per LP: [count]
- FAQ: [count] questions [✅ / ❌]
- Testimonials: [count] quotes [✅ / ❌]

💰 Cost Breakdown:
━━━━━━━━━━━━━━━━━
[N] LPs × [credits_per_lp] credits = [total_credits] credits
Your Balance: [credits_remaining]
Remaining After: [credits_remaining - total_credits]

⚠️ Credits will be deducted upon confirmation.

━━━━━━━━━━━━━━━━━━━━━
Confirm generation? [Yes / Adjust settings]
```

### Example (zh-TW style):

```
🎯 生成設定確認：
━━━━━━━━━━━━━━━━━━━━━

目標受眾：2 組
  #1: 新創公司創辦人 (9:16 直式)
  #2: 企業技術長 (16:9 橫式)

🎨 視覺設定：
- 色彩：#2fa067（主色）+ #1a1a1a（強調色）
- 字型：現代無襯線體
- 風格：科技極簡

📝 內容設定：
- 語言：zh-TW 繁體中文
- 語氣：專業但平易近人
- 手法：問題→解決方案
- CTA：「預約免費諮詢」→ https://calendly.com/your-link

📊 頁面配置：
- 每個 LP：8 頁
- FAQ：5 題 ✅
- 客戶見證：3 則 ✅

💰 費用明細：
━━━━━━━━━━━━
2 個 LP × 1,600 credits = 3,200 credits
目前餘額：44,560 credits
生成後餘額：41,360 credits

⚠️ 確認後將立即扣除 credits 並開始生成。

━━━━━━━━━━━━━━━━━━━━━
確認生成？[Yes / 調整設定]
```

### Handling User Responses

- **"Yes" / "確認" / "Go"** → Proceed to Phase 6 (Save TAs + trigger generation)
- **"Adjust [specific setting]"** → Re-present only that setting, then show updated summary
- **"Change TA"** → Go back to Phase 2
- **"Too expensive"** → Suggest cost reduction:
  - Reduce TAs
  - Use single aspect ratio
  - Reduce page count
  - Generate one LP now, others later

### Insufficient Credits

If `credits_remaining < total_credits`:
```
⚠️ Credits 不足
━━━━━━━━━━━━━━
需要：[total_credits] credits
目前餘額：[credits_remaining] credits
差額：[total_credits - credits_remaining] credits

建議方案：
A) 減少到 [X] 組 TA（符合目前餘額）
B) 使用單一 aspect ratio（費用減半）
C) 減少頁數到 6 頁（費用降低 25%）
D) 先生成最重要的 1 個 LP，其餘之後再做
E) 聯繫管理員增加 credits

Which option?
```

---

## Phase 6: Save TAs to Session (CRITICAL — only after explicit confirmation)

**Only execute this phase after the user explicitly confirms the generation config in Phase 5.**

### Step 1: Save via update_session

Save the selected TAs and all configuration into the session:

```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_ta_groups\": [<selected TA objects with ta_name, ta_description, fabt_*, spokesperson_prompt, etc.>]}"
})
```

### Step 2: Verify TAs are assigned

```
mcp_tool_call("landing_ai_mcp", "get_ta_statuses", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: [{ "ta_group_id": "ta_1", ... }]
```

Record `ta_group_id` values — these are needed for `generate_session`.

### Step 3: Confirm to user

```
✅ 設定已儲存，準備進入生成階段。
TA Group IDs: [ta_1, ta_2, ...]
Session: [session_id]

正在交接給 Landing Page 生成流程...
```

---

## Phase 7: Pass Forward

Store and pass to Phase 3 (generate-landing):
- `ta_groups`: array of selected TA configs
- `ta_group_ids`: IDs from get_ta_statuses (e.g., `["ta_1"]`)
- `aspect_ratio`: "16:9" | "9:16" | "both"
- `locale`: user's preferred language
- `brand_id`: from Phase 1
- `session_id`: from create_session (created in Phase 3 or here)
- `user_token`: JWT
- `requested_stripe_count`: from Phase 4J page count selection
- `visual_config`: color scheme, font style, visual style
- `content_config`: tone, approach, CTA text, CTA link
- `faq_data`: FAQ questions and answers (if provided)
- `testimonial_data`: testimonials (if provided)

---

## Optional: Market Research Enhancement

If user wants data-backed TA selection, invoke the `research-market` skill first:

```
→ "Want me to research market trends before finalizing your audiences?"
→ "我可以先研究市場趨勢，幫你做更有根據的 TA 選擇。要嗎？"

If yes: run research-market skill → feed insights back into TA selection
```

Key research tools:
- `google_trends_mcp` — validate TA with search volume data
- `x_mcp` — real-time sentiment for the product category
- `reddit_mcp` — community pain points and language patterns
- `tiktok_mcp` — trending content formats for younger audiences

---

## Conversation Flow Summary

```
Phase 1: generate_ta_options → AI suggests 4-6 TAs
                ↓
Phase 2: Present ALL TAs with strategic guidance → User selects
                ↓
Phase 3: Explain multi-TA implications → User confirms TA count
                ↓
Phase 4: Detailed config wizard (4A-4J):
         Aspect Ratio → Colors → Font → Visual Style →
         Language → Tone → CTA → FAQ → Testimonials → Page Count
                ↓
Phase 5: ⏸ FULL CONFIG SUMMARY + COST → EXPLICIT CONFIRMATION REQUIRED
                ↓ (only on "Yes")
Phase 6: Save TAs to session → get_ta_statuses → verify
                ↓
Phase 7: Hand off to generate-landing skill
```

**At NO point in this flow should credits be deducted without the user seeing and confirming the Phase 5 summary.**

## SaleCraft Scope & Pricing (MUST READ)

### Who We Serve
SaleCraft is for **physical product sellers only** (skincare, food, fashion, health, electronics, etc.).
- ✅ Physical products, single items, single purpose
- ❌ Software/SaaS, multi-purpose platforms, abstract services

If the user's product doesn't fit, politely redirect:
> "SaleCraft 主要服務實體產品的行銷。你的需求可能更適合其他方案。"

### Pricing — Tell Before You Act
**1 USD = 30 pts | Minimum top-up: $20 = 600 pts**

This skill costs **5-15 pts** per TA generation.

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

### After showing TA options:
```
AI 為你生成了 [N] 組目標受眾。接下來：

1. ✅ 選擇受眾（輸入編號，如 "1 和 3"）
2. ✏️ 修改某個受眾（如 "把 #2 改成針對媽媽族群"）
3. ➕ 新增自訂受眾
4. 🔍 先做市場研究再決定
```

### After TA selection + config:
```
設定完成！生成前確認：

📊 即將生成 [N] 個 LP，預估花費 [X] 點數
目前餘額：[Y] 點數

1. ✅ 確認生成（扣除點數）
2. ✏️ 調整設定（顏色/字體/風格/語言/CTA）
3. 📉 減少生成數量以節省點數
4. ❌ 取消，回到受眾選擇
```

### After generation triggered:
```
⏳ 正在生成中（約 1-3 分鐘）...

生成完成後你可以：
📱 在手機上預覽銷售頁
✏️ 編輯任何文字、圖片、版面
📸 截圖圈出要改的地方，我幫你修
🔍 優化 SEO 和搜尋引擎排名
🏠 建立完整的品牌首頁
📤 發佈到社群媒體或投放廣告
```
