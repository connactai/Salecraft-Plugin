---
name: salecraft-create
description: "Full LP creation flow — canonical 6-step Wizard (auth → session → fill → quality gate → TA → Phase 2 specs → page count → generate)"
---

# /salecraft-create — Full Landing Page Generation

Follow **CLAUDE.md → Wizard 結構 6-step**. Do NOT invent your own phase structure. Every step maps to a specific SKILL; do NOT skip steps to "save questions".

**Cost model reminder**: `create_session` + `update_session` + `analyze_brand_url` + `validate_images` + `digitize_product_text` + `generate_ta_options` + `generate_ta_spokesperson` are **ALL FREE**. Only `generate_session` deducts points. So: build session early, write progressively.

---

## Step 0 — Auth (hand user the AI Token prompt)

Required BEFORE anything else. `create_session` needs `access_token`.

```
對使用者（locale 替換）：
「① 開 https://salecraft.ai/{locale}/marketingx
  ② 點「複製 AI 登入 Token」
  ③ 貼 sc_live_... 回來給我」

拿到 → authenticate_with_token(ai_token) → access_token
```

NEVER ask for email/password.

---

## Step 1 — Create session IMMEDIATELY (placeholder names OK)

```
create_session(user_token, session_name="[LP] 新建中", brand_name="Pending", product_name="Pending")
→ session_id
```

Do NOT wait for "all answers collected". Session is the container everything else writes into.

---

## Step 2 — Fill brand data (invoke `brand-onboard` skill)

Follow `skills/brand-onboard/SKILL.md`. Key actions:

1. Ask asset source: URL / Drive / PDF / manual / none
2. URL given → `analyze_brand_url(url)` → `update_session(wizard_shared_data={...scraped...})`
3. **Field-by-field confirmation** — list what was scraped (brand_name / product_name / industry_category / description / logo / product_images / language / primary_color / social links), ask user "以上每一項對嗎?"
4. Gap-fill missing buckets one at a time

### Step 2.5 — Spokesperson (Phase 3.5 of brand-onboard)

1. `list_spokespersons(brand_id)` FIRST — if brand already has spokespersons, render via `![](photo_url)` markdown and let user pick
2. If user picks existing → `update_session(wizard_shared_data.selected_spokesperson_id=...)`, skip to Step 3
3. Otherwise offer 3 options:
   - 📸 Upload own photo → `create_spokesperson(is_ai_generated=false, photo_urls_json=[...])`
   - 🤖 AI generate → **COLLECT 9 PARAMS** (gender/age/ethnicity/glasses/build/outfit/traits/hair/extra) → `generate_ta_spokesperson` → **SHOW front_url + side_url via markdown image to user** → user approves → `create_spokesperson(is_ai_generated=true, photo_urls_json=[front, side])`
   - 🚫 No person → note preference, skip

---

## Step 3 — Quality Gate (MANDATORY if product images exist)

Run in parallel, both with `session_id`:

```
validate_images(session_id, image_urls_json=[...], industry_category, product_name, brand_name)
digitize_product_text(session_id, image_urls_json=[...], industry_category, product_name, brand_name)
```

- `overall_passed=false` → show `summary_message_zh` + translate `issue_codes` (blurry / low_res / text_unreadable / off_product) + ask per `missing_categories_labels_zh`
- OCR found "SGS/FDA/Patent" but cert bucket empty → ask user to upload cert
- User says "我知道品質會打折還是要跑" → `update_session(wizard_shared_data._quality_gate_override=true)`

Do NOT proceed to Step 4 until gate passed or override set.

---

## Step 4 — TA selection (invoke `audience-target` skill)

```
generate_ta_options(session_id, ...)  → 4-6 candidate TAs
```

**List each candidate line by line** (name / age / motivation / concerns 4 fields each). Do NOT fabricate TAs.

User picks N TAs → `update_session(wizard_ta_groups=[<flat TA objects>])`.

**STOP HERE**. Do NOT chain into aspect/page/language. Step 4 is a milestone.

---

## Step 5 — Wizard Phase 2 specs (invoke `generate-landing` skill Phase 2.9 Step 5)

Follow `skills/generate-landing/SKILL.md` Phase 2.9 Step 5 exactly.

### Step 5a — Infer pass (fill what you can from signals)

Infer from conversation / brand scrape:
- aspect_ratio ← channel signal (IG story → 9:16, Google Ads → 16:9)
- language ← **ALWAYS ASK** (multi-TA: ask per-TA; never silently pick zh-TW from brand locale alone)
- primary_color ← `analyze_brand_url`'s primary_color (per-TA)
- font_style ← industry_category typical (cosmetics → serif, SaaS → sans-serif) (per-TA)
- cta_url ← **silent default** = scraped official site URL (no user question; post-gen edit via `update_cta_link`)
- cta_text ← **silent default** = industry preset (restaurant → "立即訂位", health → "立即購買", service → "立即諮詢", general → "了解更多")
- ❌ `include_qa_section` / `include_testimonials` — **phantom fields, backend does not consume, removed 2026-04-22. Q&A and testimonials are post-gen via edit-landing's `update_faq_content` + testimonial block editing. Never write these keys to `wizard_shared_data`.**

Write inferences to session, mark `_spec_inferred_by_llm=[fields...]`:
- Shared → `wizard_shared_data`
- Per-TA → `wizard_ta_groups[i]`

### Step 5b — Ask only the truly un-inferrable (2-4 questions, no page count)

### Step 5c — Announce all inferences + let user override

```
依你前面講的，我幫你把幾個設定預填了、看看要不要改：
- 長寬比：9:16（你提過 IG）← 我幫你配
- 語言：zh-TW（官網抓到的）← 我幫你配
- ...
都 OK 就進最後一題（頁數）、要改直接講。
```

---

## Step 6 — Page count (Wizard Phase 3, last gate)

Ask ONCE. Do NOT default to 10 silently.

```
最後一題 — 頁數。每頁 200 pts、配合你內容量：
• 8 頁（1,600 pts）：活動頁 / 單品促銷
• 10 頁（2,000 pts）：一般品牌首發
• 12-14 頁（2,400-2,800 pts）：複雜體驗
• 16-21 頁（3,200-4,200 pts）：完整品牌史

你內容量大概多少？
```

User answers → `update_session(wizard_shared_data.requested_stripe_count=N)`.

---

## Step 6.5 — Cost recital (MANDATORY before generate_session)

Read session state via `get_session`, do NOT use conversation memory. Mark inferred fields with「（我幫你配）」:

```
好，幫你整理一下：
- 受眾：[tas[].ta_name]
- 頁數：N 頁 × M 組 = total
- 長寬比：9:16（我幫你配，你提過 IG）
- 語言：zh-TW（我幫你配）
- 色系：#2fa067（我幫你配，官網抓到的）
- ...
- 預計扣點：total pts（約 $USD）

餘額 X pts。有標「我幫你配」的特別看一下、要改現在講、都對就回「開始」。
```

**絕對禁止**：列出 Page 1 / Page 2 / 結構 / 每頁內容 — Architect agent 決定、你沒這資料。

---

## Step 7 — Pre-flight Self-Audit (before API call)

Follow `skills/generate-landing/SKILL.md` Phase 3 Step 0. `get_session`, run the full checklist (session fields + per-TA fields + Quality Gate status + start word). If anything missing, go back to the step that collects it.

---

## Step 8 — Fire generate_session

Only after user's explicit start word (「開始 / go / start / 執行 / 開跑」，NOT vague 「好 / OK」).

```
generate_session(session_id, ta_group_ids_json=[...], requested_stripe_count=N)
→ project_ids
```

Poll status, present results per `generate-landing` Phase 6.

---

## Cross-step state (carry, never re-ask)

`user_token`, `session_id`, `brand_id`, `ta_group_ids`, `requested_stripe_count`, `aspect_ratio`, `language` (per-TA), `primary_color` (per-TA), `cta_url`, `project_ids`, `campaign_id`.

---

## After generation

- Phase 6 — present all links (landingai.info/lp/{id}, visual editor, stitched preview) + all post-gen options (edit, crop, soft-edge, SEO, download, export, self-host, publish)
- Suggest next steps: `/salecraft-homepage` / `/salecraft-publish`

## No jargon

Never mention MCP / tokens / session_id / data_json / industry enum values / stripe_count to the user. Just do the work.

## Absolute prohibitions (LLM tends to fall into these)

- ❌ Ask TA before 素材/代言人 (Step 4 comes AFTER Step 2-3)
- ❌ Bundle TA + aspect + page + language in one batch (each Step is its own batch — do not merge across Steps)
- ❌ Treat "直接生" as permission to skip `generate_ta_options` / `validate_images` / Cost recital
- ❌ Default page count to 10 without asking
- ❌ Wait to build session — build EARLY, write progressively via `update_session`
- ❌ List per-page content (Page 1 / Page 2 structure) in Cost recital — that's Architect's job
