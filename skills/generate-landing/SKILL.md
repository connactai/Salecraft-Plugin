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

### ⚠️ Pre-flight gotchas

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

## Phase 2.85: Image Sufficiency Scan (MANDATORY — 在 Phase 2.9 前執行)

**LLM 最愛跳過這關。** 不要「再三口頭確認」就算數——**實際呼叫 `get_session` 掃 session state**，親眼確認四個 asset 桶有什麼。使用者 5 輪前對話講的「我有產品圖」，不代表真的有上傳進系統。你的工作是在扣點之前抓到這個落差，否則使用者會拿到一份 AI 瞎編的 LP。

### 掃描動作

進 Phase 2.9 之前，呼叫 `get_session` 檢查：

```
mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token,
  "session_id": session_id
})
```

檢查四個 asset 桶：

| Asset 類別 | 在 session 的路徑 | 空 = 什麼問題 |
|---|---|---|
| 產品圖 | `wizard_shared_data.product_images` (array) | AI 從文字瞎想產品樣子，常跟實品差很遠 |
| Logo | `wizard_shared_files.logo_image` (string) | AI 自己配一個 logo，日後很難換 |
| 代言人 | `wizard_shared_data.spokesperson_faces` (array) | AI 生成虛擬人臉，法律風險 + 無法真實行銷 |
| 認證 / 檢驗 | `wizard_shared_data.certification_images` (array) | Factory 的 CERTIFICATE GUARD 會跳過所有認證 badge |

### 若四個桶**全空**（純文字敘述）

**必須停下來問**，即使使用者已經說「你直接跑就好」：

> 「先打住——我掃了你的 session 資料，**四個素材桶（產品圖 / logo / 代言人 / 認證）全部都沒有實際圖片，全是文字敘述**。
>
> 這代表 AI 會自己想像：
>   - **產品樣子**（可能跟實品差很遠，之後截圖給客戶一看就露餡）
>   - **Logo**（AI 瞎編一個，跟你真實品牌識別完全無關）
>   - **代言人**（AI 生一張虛擬臉，不能用在真實行銷、還有法律風險）
>   - **認證 / 檢驗圖**（沒提供就不會出現任何認證 badge，信任感下降）
>
> 你手邊有以下任何一項嗎？
>   1. 產品實拍（手機隨便拍都可以）
>   2. 品牌 logo 圖檔
>   3. 代言人照片
>   4. 認證 / 專利 / 檢驗報告
>   5. 公司/產品網站連結（我可以自動抓）
>   6. Google Drive 連結（可批次匯入）
>
> 如果真的都沒有、就是要跑 **AI 想像版**——OK，但我需要你明確回『我知道這是 AI 想像版，還是要跑』才會繼續。」

**除非下列任一成立，否則不准進 Phase 2.9**：
- 至少有 1 張真實圖片進入任一桶，或
- 使用者明確書面同意「AI 想像版、繼續跑」

### 若部分有、部分空

不要只問「確認沒有 logo 喔？」——**針對空的桶逐個問**：

> 「我看到你有產品圖（很好），但沒有 logo / 代言人 / 認證。這幾項是打算 AI 生，還是你手邊有可以傳給我？」

等使用者對每個缺的桶做出明確選擇（自備 or AI 生），再進 Phase 2.9。

### ⚠️ 反模式

- ❌ 只在對話裡「再三確認」，沒實際呼叫 `get_session` 掃資料
- ❌ 假設使用者前面答過就等於有上傳（使用者可能只是嘴巴講「有 logo」但沒實際丟檔）
- ❌ 只掃一個桶（如只檢查 product_images），忽略其他三個
- ❌ 發現全空但使用者態度積極就跳過警告；**態度不等於授權**，要明確書面同意

### ⚠️ Phase 3.9 Quality Gate 回溯檢查（backup — 只在 primary gate 沒跑時觸發）

Phase 2.85 掃完**有沒有**圖之後，還要檢查那些圖**品質過不過關**。

**執行決策（照順序判斷、第一個滿足就停止）**：

```
session_data = get_session(session_id)
product_images = wizard_shared_data.product_images + wizard_shared_files.product_images

# Case A: 沒任何產品圖 → primary gate 本來就該 skip，這裡也 skip
if product_images 是空:
    略過 backup gate、直接進 Phase 2.9
    Phase 2.85 image-sufficiency 警告已經涵蓋這個 case

# Case B: primary gate 已跑過（正常流程：brand-onboard Phase 3.9）
elif session.image_censor_results 最新一筆 overall_passed == true
     OR (最新一筆 overall_passed == false AND session.wizard_shared_data._quality_gate_override == true):
    略過 backup gate（使用者已經在 brand-onboard 看過、要嘛通過要嘛明確授權繼續）
    直接進 Phase 2.9

# Case C: primary gate 從未跑、或從未看到 overall_passed=true 的紀錄
# （例：使用者從舊 session 進來、別的 AI 建的 session、繞過 brand-onboard）
else:
    **立刻 call** validate_images + digitize_product_text（兩個都帶 session_id）
    把 summary_message_zh + missing_categories_labels_zh 給使用者看
    overall_passed == false → 等明確書面同意（接受詞同 brand-onboard Phase 3.9）才進 Phase 2.9
```

**不要無條件重跑**。若 primary gate 已成功、這裡**必須**跳過、不要浪費 28 秒跟使用者的 token 再跑一次。Backup 的角色是**兜底**，不是「雙重保險」。

使用者在 brand-onboard Phase 3.9 回了「我知道品質會打折、還是要跑」時，brand-onboard 會把 `session.wizard_shared_data._quality_gate_override = true` 寫進 session（實作上當下 call `update_session` 把這個 flag 加進去）。這裡讀到 flag 就尊重、不再問第二次。

---

## Phase 2.9: Pre-Generation spec confirmation（Wizard Phase 2 完成 + Phase 3 開始；不是 interrogation）

> **Backend phase 對應**：Step 5 的 aspect_ratio / **language** / color / font 屬 **Wizard Phase 2**（這四項 generation-time pipeline 會實際讀）。`cta_text` / `cta_url` 技術上也在 `wizard_shared_data` 但**此處用 silent default、由 edit-landing `update_cta_link` 後製微調**。**不要問 Q&A / 客戶見證**——那是 **post-gen** 的事（`update_faq_content` / testimonial blocks 編輯），backend 生成 pipeline 從不消費 `include_qa_section` / `include_testimonials` 欄位（grep 2026-04-22 確認 backend 0 hits、這兩個是 plugin 殘留的 phantom fields、寫進 session = silent JSONB drop）。Step 6 的**頁數** (`requested_stripe_count`) 是 **Wizard Phase 3**（這個 phase 只有這一項、完成即觸發生成）。別把頁數問題混到 Phase 2 裡、也別把語言拖到 Phase 3。


### ❌ 反模式：把規格題目串成一張大表單逐題問

LLM 容易把 Phase 2.9 跑成逐題 interrogation、使用者像填表單。**這不是 SaleCraft 的生成 LP 路線**——session 裡已經有的**不要再問**、能推斷的就 infer-then-announce、只補真的缺的。

### ✅ 正確心智模型：讀 session 狀態、只補缺的、立刻扣點

到 generate-landing 被觸發時，session 裡**應該已經有**以下（來自 brand-onboard Step 2-3 + audience-target Step 4）：

| 欄位 | 來源 | 若缺少 |
|------|------|--------|
| `wizard_shared_data.brand_name / product_name / industry_category / base_description` | brand-onboard Step 2 | 回 brand-onboard |
| `wizard_shared_files.logo_image / product_images` | brand-onboard Step 2 | 回 brand-onboard |
| `wizard_shared_data.spokesperson_choice` + params | brand-onboard Phase 3.5 | 回 brand-onboard |
| `session.image_censor_results[-1].overall_passed == true`（或 `_quality_gate_override=true`）| brand-onboard Phase 3.9 | 回 brand-onboard |
| `session.wizard_shared_data.product_text_model` | brand-onboard Phase 3.9 | 回 brand-onboard |
| `session.wizard_ta_groups[]` ≥ 1 組 | audience-target Step 4 | 回 audience-target |

**你在這裡的 scope 只有 Step 5-6 的欄位**：aspect_ratio / language / primary_color / font_style / requested_stripe_count。`cta_url` 自動用 brand 官網、`cta_text` 自動用產業預設（例：餐飲 → "立即訂位"）、使用者不需手動選、post-gen 可用 `update_cta_link` 微調。**Q&A / 客戶見證 不在此 scope**（phantom fields、backend 不讀、全屬 post-gen 走 edit-landing）。其他都是前面 skill 的事、**不准在這裡重新問**。

### Step 0 — 讀 session、列出還缺什麼

```python
session = get_session(session_id)
shared  = session["wizard_shared_data"]
tas     = session["wizard_ta_groups"]

# 前置條件檢查（缺就請使用者回去跑對應 skill、不准自己補、不准跳關到 Step 5）
missing_prereq = []
if not shared.get("brand_name"):             missing_prereq.append("brand 基本資訊 — brand-onboard Step 2 還沒跑")
# Phase 3.5 代言人必跑、2026-04-22 觀察到 AI 從 Phase 1 直跳 TA/Step 5、完全沒問代言人
if "spokesperson_choice" not in shared:      missing_prereq.append("代言人 — brand-onboard Phase 3.5 還沒跑（先 list_spokespersons 看既有、再問使用者挑既有 / AI 生 / 不放）")
if not session.get("image_censor_results") and not shared.get("_quality_gate_skipped_no_images"):
                                             missing_prereq.append("Quality Gate — brand-onboard Phase 3.9 還沒跑")
if not tas:                                  missing_prereq.append("TA — audience-target Step 4 還沒跑")
if missing_prereq:
    # 告訴使用者缺什麼、回去跑對應 skill、這邊不動
    return

# Step 5-6 自己 scope 的欄位，缺就處理（Q&A / 見證是 phantom fields、不列、post-gen 走 edit-landing）
needs = []
if not shared.get("aspect_ratio"):     needs.append("aspect_ratio")
if not shared.get("language"):         needs.append("language")  # 明確問、見下方 Step 5 語言紀律
if not shared.get("primary_color") and not shared.get("color_mood_keyword"):
                                       needs.append("color")
if not shared.get("font_style"):       needs.append("font_style")
# CTA：不問使用者、silent default = brand 官網 URL + 產業預設 text、post-gen 走 update_cta_link
if not shared.get("cta_url"):
    shared_cta_url = shared.get("website_url") or shared.get("brand_url") or "#"
    # 這裡只在 session 寫 default、不加進 needs（不問使用者）
if not shared.get("cta_text"):
    # 產業預設：餐飲→"立即訂位"、保健食品→"立即購買"、服務→"立即諮詢"、general→"了解更多"
    ...
# requested_stripe_count 永遠最後問（不管其他欄位狀態）
```

### Step 5 — **infer 能推的、剩下才問**（**不包含頁數**）

**核心 meta-rule**（CLAUDE.md #6.5 NO SILENT DEFAULTS）：`needs` 清單裡每一項都得被「碰到」——要嘛使用者親答、要嘛 LLM 推斷 + 在對話裡宣告讓使用者有機會反對。**絕對禁止**把欄位留空讓 backend 吃預設。

#### 🔴 Per-TA vs Shared — 多 TA 時不同 TA 會有不同設定（這是故意的）

Backend 把 Phase 2 spec 分成兩類儲存：

| 欄位 | 儲存位置 | 多 TA 時 |
|------|----------|---------|
| `visual_style`（風格）| `wizard_ta_groups[i].visual_style` | **每 TA 各自設**，可以 A 走優雅 / B 走俏皮 |
| `theme` | `wizard_ta_groups[i].theme` | **每 TA 各自設** |
| `primary_color`（色系）| `wizard_ta_groups[i].primary_color` | **每 TA 各自設**，可以 A 墨綠 / B 玫瑰金 |
| `language` | `wizard_ta_groups[i].language` | **每 TA 各自設**，可以 A=zh-TW / B=en |
| `spokesperson_prompt`（代言人）| `wizard_ta_groups[i].spokesperson_prompt` | **每 TA 各自挑**（`generate_ta_options` 每組回傳 `spokesperson_prompts[]` 2 個候選、使用者 per-TA 挑一個、或挑「都不用」寫 `null`）。LP hero 首屏視覺關鍵、不准忽略 |
| `spokesperson_faces`（代言人上傳照片）| `wizard_ta_group_files[i].spokesperson_faces[]` | **每 TA 各自設**。若使用者選自己上傳照片、走 `create_spokesperson` 拿 URL 寫進這裡 |
| `aspect_ratio`（長寬比）| `wizard_shared_data.aspect_ratio` | shared（所有 TA 同一份）|
| `cta_url` / `cta_text` | `wizard_shared_data.cta_text/url` | shared（silent default、不問使用者、post-gen 走 `update_cta_link`）|
| `requested_stripe_count`（頁數）| `wizard_shared_data.requested_stripe_count` | **shared（這個 session 內所有 TA 同樣頁數）**。若使用者要不同 TA 不同頁數（例：TA1 8 頁、TA2 12 頁）、**單一 session 做不到**——必須跟使用者講清楚「在同一次生成裡每組頁數要相同、想要不同頁數要分兩次 session」、不要假裝支援、也不要偷偷挑一個數字 |

**實作**：`num_tas = len(wizard_ta_groups)`，
- 若 `num_tas == 1`：per-TA 欄位可以跟 shared 合併問（1 個 TA 沒差異）
- 若 `num_tas >= 2`：**per-TA 欄位必須逐 TA 問或逐 TA 推斷**，不要把 TA-A 的 primary_color 套到 TA-B

**範例（num_tas=2、兩個 TA 分別是「商務宴客 B2B」和「家庭聚餐 B2C」）**：

```
你選了兩組受眾，每組可以有自己的風格：

**🎯 TA-A 商務宴客 B2B**
- 語言：英文（你前面講過這組主打海外 B2B）← 我幫你配
- 色系：深咖啡 + 金（商務高端）← 我幫你配
- 字體：襯線（editorial 調性）← 我幫你配

**🎯 TA-B 家庭聚餐 B2C**
- 語言：繁中 ← 我幫你配
- 色系：暖綠 + 乳白（親切溫暖）← 我幫你配
- 字體：手寫（溫暖情感）← 我幫你配

（兩組共用的設定：9:16 直版、CTA 連 LINE、含 Q&A 區塊、不含見證）

兩組的風格都 OK 嗎？有要改哪組的哪項直接講。
```

#### Step 5a — 先跑一次 infer pass、推能推的

進 Step 5 批量問之前，**先在心裡過一遍 `needs`**，檢查下列 signal 能不能推出該欄位的合理值：

| 欄位 | 可從這些 signal 推 | 推斷範例 |
|------|-------------------|---------|
| `aspect_ratio` | saleskit / brand-onboard 對話提到渠道（IG 限時 / TikTok / 桌機官網 / Google Ads） | 提過「IG 限時」→ `9:16`；提過「官網 hero」→ `16:9`；沒提或兩邊都要 → 預設兩邊 |
| `language` | 使用者對話主要語言 + brand scrape language + TA ta_description 語言 | 使用者講繁中、brand 官網繁中 → `zh-TW`；若 TA 中某組 ta_description 是英文 → 該 TA 推 `en`、其他推 zh-TW |
| `primary_color` | `analyze_brand_url` 抓的 primary_color / logo 主色 | 抓到 `#2fa067` → 直接用；沒抓到 → 才問 |
| `font_style` | `industry_category` 的通用字體偏好 | cosmetics / jewelry / luxury → `serif`；software / electronics / SaaS → `sans-serif`；handmade / artisan → `handwritten` |
| `cta_url` | brand-onboard 抓的官網 URL | **永遠 silent default**、不問使用者、post-gen 走 `update_cta_link` 微調 |
| `cta_text` | 產業類別 default | **永遠 silent default**（餐飲→「立即訂位」、保健→「立即購買」、服務→「立即諮詢」、general→「了解更多」）、post-gen 可改 |

**推斷後寫進 session**（update_session）並把 **`_spec_inferred_by_llm` 加進 session**（flag 起來、Cost 複誦時會標記）。**注意 per-TA 欄位要寫進對應的 `wizard_ta_groups[i]` 條目、不要寫進 `wizard_shared_data`**：

```
# 寫 shared 欄位（所有 TA 共用）
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token, "session_id": session_id,
  "data_json": '{"wizard_shared_data": {
    "aspect_ratio": "9:16",
    "cta_url": "https://example.com",
    "cta_text": "立即訂位",
    "_spec_inferred_by_llm": ["aspect_ratio", "cta_url", "cta_text"]
  }}'
})

# 寫 per-TA 欄位（每個 TA 各自寫）
# TA 的完整 wizard_ta_groups 條目必須完整包含所有原欄位（id / ta_name / ta_description / ...），
# 加入 language / primary_color / visual_style 等 per-TA spec
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token, "session_id": session_id,
  "data": {"wizard_ta_groups": [
    {  # TA-A：商務 B2B 英文優雅
      "id": "ta_1", "ta_group_id": "ta_1",
      "ta_name": "Business Entertainment Hosts",
      "ta_description": "...",
      "language": "en",
      "primary_color": "#3b2a1f",
      "visual_style": "serif_editorial",
      "theme": "luxe",
      "_spec_inferred_fields": ["language", "primary_color", "visual_style"]
    },
    {  # TA-B：家庭 B2C 繁中溫暖
      "id": "ta_2", "ta_group_id": "ta_2",
      "ta_name": "Family Gathering Hosts",
      "ta_description": "...",
      "language": "zh-TW",
      "primary_color": "#a3b18a",
      "visual_style": "handwritten_warm",
      "theme": "warm",
      "_spec_inferred_fields": ["language", "primary_color", "visual_style"]
    }
  ]}
})
```

#### 🔴 Step 5a 推斷紀律 — 沒 signal 不准 infer

「我幫你配」不是免費通行證。Step 5a 推斷表（line 377-385）的每個 signal 都是**條件式**——條件沒滿足 → 推到 Step 5b 問、禁止 silent 填。

**規則**：

- 合法 infer：推斷值必須引用具體 signal source（scrape 欄位 / 使用者講過的句子 / 產業預設）、Cost 複誦必附來源「**9:16 直版**（你提過 IG 限時）」
- 非法 infer：signal 不存在就填 = 違規（例：餐廳產業 = 16:9 是假 signal、沒這種產業預設）
- 混列合規與違規：批裡只要 1 項沒 signal、整批退回 Step 5b 問
- 自創不在表裡的欄位：`cta_text` / `tagline` / `tone` 等一律問使用者、禁止 AI 編文案值
- `include_qa_section` / `include_testimonials` 是 phantom fields（backend 0 hits）、禁止寫入、Q&A / 見證 post-gen 走 edit-landing

**自檢**：每個「（我幫你配）」若被使用者問「為什麼是這值？」能指到 session 欄位 / 對話某句 / 產業預設嗎？指不到 = 腦補、重寫。

#### 🔴 語言（language）紀律

**語言 signal 特別弱**：「使用者跟我用繁中 + brand 是台灣」不夠 silent-infer（台灣品牌可做英文版打海外、多 TA 最常見分流就是 zh-TW vs en、同語言配兩組 = 抹掉分流意義）。

**規則**：

- **單 TA**：對話明確提過目標語言 → 可推、Cost 複誦必附 signal。未提 → 明確問「要做哪個語言？（9 locale 清單）」
- **多 TA（≥2）**：**一律逐組明確問**、禁止 silent 配同語言
- 禁止把多組 `language` 填同一值當「（我幫你配）」
- 禁止以「brand 官網語言」當 signal 推 language（官網語言 ≠ 目標市場語言）

**多 TA 正確問法範本**：

> 「你兩組 TA 可以各自跑不同語言：
> - TA1 [ta_name]：要繁中還是英文？
> - TA2 [ta_name]：這組看起來有國際客群的調性、要跑英文版嗎？」

#### Step 5b — 只問剩下真的推不出來的

Infer pass 跑完、`needs` 通常從 7-8 題縮到 0-3 題。**剩下的才問使用者**、仍然 2-4 題一組。

#### Step 5c — 宣告推斷值、等使用者反對

Step 5a/5b 跑完、進 Step 6 頁數之前，把**所有推斷的欄位一次宣告**給使用者、給反對機會：

```
依你前面講的，我幫你把幾個設定預填了，看看要不要改：

- 長寬比：**9:16 直版**（因為你提過 IG 限時動態）
- 色系：**墨綠 #2fa067**（官網抓到的品牌主色）
- 字體：**襯線**（你做保健食品、襯線比較合調性）
- CTA 連結：**你的官網**（你提過的那個）
- Q&A 區塊：**加**（保健食品猶豫型客人多、Q&A 轉換有幫助）
- 客戶見證區塊：**不加**（你沒有實際評價、加了會是空的）

都 OK 就進最後一題（頁數），有要改直接講哪項。
```

使用者說「都 OK」或「改 X」→ 該反對的就 update_session 改、其他保留、然後進 Step 6。

### Step 5 原則：**讓使用者只需要點頭或否決、不要從空白答每題**

問法：**用人話、給選項、不要一次丟太多**。**絕對不要照 gate 編號 1-12 念給使用者聽**——那是內部 audit list。

問的範例（使用者只缺 aspect / language / cta）：

```
再三題就開工：

1. LP 主要在哪裡看？
   ① 手機直版（9:16，IG 限時 / TikTok 適合）
   ② 桌機橫版（16:9，官網 / Google Ads 適合）
   ③ 兩邊都要（預設）

2. 主要語言？（給當下推薦的 2-3 個，不要列全部 15 個）

3. 最下面的行動按鈕要連去哪？
   ① 官網 ② 購買頁 ③ LINE ④ 預約頁 ⑤ 先不填
```

若 `needs` 某項 session 早已寫過（例：brand-onboard 爬官網時 `primary_color` 已抓到）→ **不要重問**。讀到什麼、略過什麼。

**每組答完立刻 `update_session` 寫進去 + `get_session` 驗寫入**（protocol 見下方「每組答完就寫進 backend」段）。

#### 🔴 Step 5 寫入驗證（2026-04-22 強化）— 禁止「嘴上說寫了」

每次 `update_session` 寫任何 Step 5 spec 欄位（aspect_ratio / language / primary_color / font_style / visual_style / cta_url / cta_text）後、**立刻 `get_session` 讀回 + 針對該欄位 assert**、禁止繼續往下走直到驗證通過。

**Per-TA 欄位驗證**（language / primary_color / visual_style 等）：

```python
update_session(wizard_ta_groups=[{"id": "ta_2", "language": "en", ...}])
s = get_session(session_id)
ta_2 = next((t for t in s["wizard_ta_groups"] if t.get("id") == "ta_2"), None)
assert ta_2 and ta_2.get("language") == "en", (
    f"❌ language 寫入失敗 on ta_2. Got: {ta_2.get('language') if ta_2 else 'TA not found'}. "
    f"要重試 update_session 或檢查 payload shape"
)
```

**Shared 欄位驗證**（aspect_ratio / cta_url 等）：

```python
s = get_session(session_id)
shared = s["wizard_shared_data"]
assert shared.get("aspect_ratio") == "16:9", f"❌ aspect_ratio 寫入失敗: got {shared.get('aspect_ratio')}"
```

**驗證失敗 = 立刻停、不准進 Cost 複誦、不准 `generate_session`**。常見失敗原因：

- Per-TA 欄位寫進 `wizard_shared_data` 而非 `wizard_ta_groups[i]`（shared vs per-TA 搞錯）
- Payload 用 nested `ta_data` shape、被 backend `_flatten_ta_entry` 時意外 drop 欄位
- `update_session` 回 200 但 body 沒 echo 新值（backend silent skip）

**為什麼這關關鍵**：2026-04-22 觀察到 AI Cost 複誦宣稱「TA2 language=en」、但 DB log 從沒出現「language changed」記錄、Wizard UI 顯示語言欄位空白——plugin 根本**沒寫成功**、產出來的 TA2 LP 整份是中文。此關防止 AI 嘴上說寫了但實際沒打 API 的「假確認」。

### Step 6 — 最後問頁數、立刻 Cost 複誦

前面 spec 都到位才問頁數。頁數一答完 → `update_session` 寫 `requested_stripe_count` → 立刻算 cost → 進 Cost 複誦。

```
最後一題——**頁數**。
每頁 200 pts，配合你內容量：
• 8 頁（1,600 pts）：活動頁 / 單品促銷
• 10 頁（2,000 pts）：一般品牌首發（預設）
• 12-14 頁（2,400-2,800 pts）：複雜體驗 / 多面向
• 16-21 頁（3,200-4,200 pts）：完整品牌史 / 多產品線

你內容量大概多少？
```

### ⚠️ 絕對不准做的事

- ❌ 把 Wizard Step 2-6 的 gate 逐條照念給使用者聽。**那是內部 audit 清單**（check session 欄位存在與否）、**不是對使用者的問題清單**
- ❌ 在這裡重問素材 / logo / 產品圖 / 代言人（brand-onboard Step 2-3 已處理）
- ❌ 在這裡重選 TA（audience-target Step 4 的事）
- ❌ 把頁數塞中間。頁數永遠是最後
- ❌ 一次丟 7-8 題讓使用者填完。一組 2-4 題、答完再下一組

### 🔴 MANDATORY: 每批答完就 `update_session` 寫進 backend（不要只在對話裡記）

**痛點**：LLM 問完 gates 只在對話 context 記答案、沒寫回 session。若對話太長 context 被裁掉、使用者換 AI、session 被另一個 AI 接手 → 答案全失。使用者以為「已登記」結果 backend session 還是空的。**這會在 Phase 3 cost 複誦時露餡，或更糟是 generate_session 用預設值導致使用者預期跟實際 LP 不符**。

**強制規則**：使用者答完一批（3-4 題）後，**立刻**呼叫 `update_session` 把那批答案寫進 `wizard_shared_data`（共用）或對應的 TA 條目（`wizard_ta_groups`），再進下一批。

#### 🔴 4-batch 結構（對應 Wizard 6-step）

每個 Step 的題目**獨立一批**、答完就 `update_session`、**不要跨 Step 合併**。

##### 批 1 — Step 2 素材 / 代言人寫入範例

素材組多數在 `brand-onboard` 跑過了（logo / 產品圖 / 認證桶）。這批 update 通常只補代言人選擇：

```
# 使用者選了「AI 生成代言人」+ 9 題參數
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_shared_data\": {\"spokesperson_choice\": \"ai_generated\", \"spokesperson_params\": {...9 params...}}}"
})
# 若選「不使用人物」寫 "spokesperson_choice": "none"
# 若選「上傳自己照片」則 create_spokesperson 已處理、不需再 update_session
```

**→ 批 1 答完：跑 Step 3 Phase 3.9 Quality Gate（validate_images + digitize_product_text）再進批 2**

##### 批 2 — Step 4 TA（單題）寫入範例

TA 是獨立里程碑、**只有一題**、答完就停、**不要接著問長寬比 / 頁數 / 語言**（那是批 3 的事）：

```
# 使用者從 generate_ta_options 候選裡挑了 1-3 組 TA
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data": {"wizard_ta_groups": [<被挑中的 TA objects、flat 結構含 id / ta_group_id / ta_name / ta_description>]}
})
```

##### 批 3 — Step 5 Wizard Phase 2 spec 寫入範例

**不含頁數**。shared 欄位 + per-TA 欄位分開寫：

```
# 寫 shared（所有 TA 共用）
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data": {"wizard_shared_data": {
    "aspect_ratio": "9:16",
    "cta_url": "https://line.me/R/...",
    "cta_text": "立即預約"
    // 不寫 include_qa_section / include_testimonials — phantom fields、backend silent drop
  }}
})

# 寫 per-TA（多 TA 時每組可不同：語言 / 色系 / 字體視覺風格）
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token, "session_id": session_id,
  "data": {"wizard_ta_groups": [
    {"id": "ta_1", "ta_group_id": "ta_1", "ta_name": "...",
     "language": "en",
     "primary_color": "#2fa067",
     "visual_style": "serif_editorial"},
    ...
  ]}
})
```

##### 批 4 — Step 6 Wizard Phase 3 頁數（單題）寫入範例

**最後一題**、答完立刻 Cost 複誦：

```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data": {"wizard_shared_data": {"requested_stripe_count": 10}}
})
# 立刻進 Cost 複誦、等啟動詞
```

#### 批寫入完成後驗證（🔴 MANDATORY、違反 = 使用者投訴）

每批 update 完、**一定要 `get_session` 讀一次、逐 key assert 剛才寫的欄位真的在**：

```python
# 剛才 update_session 寫的 key 列出來（這個 list 要精準對應 update payload）
keys_i_just_wrote = ["aspect_ratio", "language", "primary_color", "cta_url", "cta_text", ...]

session_state = mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token, "session_id": session_id
})
shared = session_state["wizard_shared_data"] or {}

missing = [k for k in keys_i_just_wrote if k not in shared or shared[k] is None]
if missing:
    # ❌ 那批 update 實際 silent-dropped — 很可能寫錯了 nesting（頂層 vs wizard_shared_data）
    # 立刻檢查 nesting + 重寫 + 再驗
    raise WriteVerificationFailed(f"silently dropped: {missing}")
```

**⚠️ 三種「假驗證」絕對不算**（LLM 常誤以為這些 = 成功、實際 backend silently dropped unknown key）：
- ❌ 只看 `update_session` 沒 raise exception — **不算**（backend 200 OK、但 key drop）
- ❌ 只看回傳 `updated_at` 變了 — **不算**（unknown key 被 drop 時 updated_at 也會動）
- ❌ 只看回傳 `success: true` — **不算**

**唯一算數的驗證**：`get_session` → **對你剛寫的每個 key 存在性 assert**。少一個就是 drop 了、立刻重寫。

**Brand 欄位的 nesting 規則**（最容易踩、見 CLAUDE.md 6.5 白名單）：
- `brand_name` / `base_description` / `value_proposition` / `brand_story` / `tagline` / `primary_color` / `key_features` / `cuisine_type` / `signature_dishes` / `operating_hours` / `pricing_info` / `target_audience` / `trust_certifications` 等
- **必須 nest 在 `wizard_shared_data`**、寫成頂層 key 會被 backend silently drop
- Drop 之後 backend 仍回 200 + `updated_at` bumped、所以**沒做 key-level assert 完全發現不了**
- 漏寫的後果：生 LP 時 Strategist 拿不到這些欄位、走預設品牌敘事、使用者確認過的內容 0 影響

這個讀回驗證步驟不可省。使用者「已登記」的感覺必須是 backend 真的有寫。

### 最後 Cost 複誦 + 啟動確認（強制範本）

所有 gate 問完、每批 `update_session` 寫進去、也每批 `get_session` 讀回驗證過之後，**再做一次 `get_session` 把所有欄位撈出來當複誦基準**——**不准用你對話記憶裡的版本**。

```
# 複誦前最後一次讀 session
session_state = mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token, "session_id": session_id
})
# 所有下列欄位全部從 session_state 取，不要自己編
shared = session_state["wizard_shared_data"]
tas    = session_state["wizard_ta_groups"]
```

**只複誦使用者輸入的規格、不要編 stripe 結構**：

**TA 名稱必須列完整 `ta_name` 字串**（例「TA1 — 追求極致儀式感的高淨值慶生者」）、**禁止縮寫成 A/B/甲/乙 或只列代號**——使用者記的是名字、不是編號，縮寫 = 使用者無法 double-check 認對組。

```
好，幫你整理一下（LLM 推斷的欄位後面加「（我幫你配）」、使用者親答的不加、讓他知道哪些要 double-check）：

- 受眾（逐組列完整 ta_name、不縮寫）：
    - TA1 — [tas[0].ta_name 完整字串]
    - TA2 — [tas[1].ta_name 完整字串]
    - ...
- 頁數：[shared.requested_stripe_count] 頁 × [len(tas)] 組 = [total_stripes] 頁
- 長寬比：[shared.aspect_ratio]{若 "aspect_ratio" in shared._spec_inferred_by_llm 加「（我幫你配，因為你提過 X）」}
- 語言（per-TA、逐組列）：
    - TA1：[tas[0].language]{若推斷加 signal citation}
    - TA2：[tas[1].language]{同上}
- 色系（per-TA、逐組列）：
    - TA1：[tas[0].primary_color]{同上}
    - TA2：[tas[1].primary_color]{同上}
- 字體：[shared.font_style 或 per-TA]{同上}
- CTA：[shared.cta_text] → [shared.cta_url]（silent default、post-gen 可用 update_cta_link 改）
  — 這行不需要使用者 double-check、因為是 silent default、不算「我幫你配」
- 代言人（per-TA、逐組列、禁止省略）：
    - TA1 [ta_name]：[spokesperson_prompt 描述 或「不用人物」]{若推斷加標記}
    - TA2 [ta_name]：[spokesperson_prompt 描述 或「不用人物」]{若推斷加標記}
    - …（每組都列、不准合併成一句 shared）
- 預計扣點：[stripe_cost × stripe_count × num_tas] pts（約 $[USD]）
  （代言人：0 pts——`generate_ta_spokesperson` 走帳號免費配額、已在 Phase 2.5 消耗、不重複扣；代言人在 LP 實際出現的成本包在 stripe_cost 裡）
  **扣點機制（backend 實際行為，要講清楚）**：
    先預扣 [stripe_cost × requested_pages × num_tas] pts（per TA 各自預扣 requested × 200）、
    生完看實際頁數：多生不加收、少生會退差額（`stripe_adjustment`）。
    例：預扣 2,400 pts（12 頁 × 200）、實際生 11 頁 → 退 200、net 2,200 pts。
    你看 transaction log 的 `-2000` 那種數字是**預扣**、不是 net。

你目前餘額 [X] pts。**有標「（我幫你配）」的欄位特別看一下、要改現在講**；都對就回「開始」我就跑；回「取消」就先不動。
```

### 為什麼要標推斷欄位

根據 CLAUDE.md #6.5 **NO SILENT DEFAULTS**，使用者親口答 vs LLM 推斷是**不同信心度**——使用者親答的錯了是使用者責任、LLM 推斷的錯了是 LLM 責任。把推斷欄位在複誦時標出來：
1. 使用者看到就會多看一眼、發現推錯當場改
2. 不會生完 LP 才發現「我沒講過要 9:16 啊怎麼出直版」→ 退費投訴
3. 使用者親答的欄位（例：頁數）不標、不干擾閱讀

### 🔴 絕對禁止 — 複誦時不要列每頁內容

**反模式**：

```
❌ — 結構 —
❌ Page 1: Hero — A love letter from Taiwan to the world
❌ Page 2: The Positioning — world-class technique × Taiwanese terroir
❌ Page 3: The Culinary Philosophy — 和食 × 歐陸 × 著時食材
❌ ...（編 8 頁）
```

每頁的標題、副標、body、視覺、brightness、bg 描述，全部是 **Architect agent 在 generate_session 之後才決定的**。你**現在還沒呼叫 generate_session**，你根本沒有這些資料。你列出來的那 8 頁是你自己編的小說、**不是 LP 真正會長的樣子**。

這等同於 `CLAUDE.md` 最上面 **EXECUTION DISCIPLINE** 段警告的「impersonate backend agents」失誤——你用策略文取代 API call、害使用者看到假的預期、實際 LP 生出來不一樣就退費投訴。

**複誦只能包含**：使用者**明確回答過的規格**（TA / 頁數 / 長寬比 / 語言 / 色系 / 字體 / CTA / Q&A / 見證）+ **該規格對應的費用**。

**不准包含**：
- ❌ Page 1 / Page 2 / Page N 具體標題
- ❌ 每頁的內容主軸、副標題、body 文案
- ❌ 視覺 / 色調 / brightness 變化
- ❌ 素材如何被用（「所有菜品、空間、人物視覺:AI 生成」這種 stripe-level 決策）
- ❌ 「文字敘事:從官網中文內容翻譯改寫成英文 editorial 語氣」這種 Architect 決策

若使用者追問「那實際每頁長怎樣？」→ 老實回「那是 AI 生成階段才會決定的、我現在沒有這個資料、生完才能看到。你要先看草稿再生嗎？我們目前沒有『只產結構不扣點』的預覽模式，只能整份生。」

### 啟動詞白名單

- **接受**：開始 / go / 執行 / 開跑 / start / do it / 跑吧 / 動手
- **拒絕**（模糊，要再確認一次）：好 / OK / 嗯 / 可以 / alright / sure / 應該可以

遇到拒絕類回覆，**不要自行解讀為同意**，再問一次：「確認要『開始』嗎？點數會立刻扣。」

### 「直接生」模式的正確解讀（LLM 常誤解）

使用者說「直接生 / 直接跑 / 不要問那麼多 / 用預設就好」時，**不是**「跳過所有 wizard、LLM 自己猜一套 spec 去扣點」。正確解讀是「**我接受 LLM 推斷的值、不用每題問我、但工具該 call 還是要 call**」。

#### ❌ 錯誤解讀

```
使用者：「直接生就好、不要問太多」
LLM：「OK，10 頁 × 1 組 TA × 橫版 × 繁中、2000 pts、我跑了」
      ↓
LLM 實際行為：
  ❌ 沒 call generate_ta_options（TA 是 LLM 幻想的、不是後端產的候選）
  ❌ 沒 call validate_images（Quality Gate 跳過）
  ❌ aspect_ratio / language 直接當預設、沒宣告 inference 來源
  ❌ 頁數「10」是 LLM 自己定的、沒問使用者
  ❌ Cost 複誦只列結果、沒列 LLM 推斷了哪些欄位
```

這會生出一份**使用者完全不知道自己同意了什麼的 LP**、爛了就退費。

#### ✅ 正確「直接生」模式（最短合規路徑、大約 4 次使用者互動）

即使使用者要「不多問」，**仍必須跑的 backend 工具 + 使用者互動**：

```
使用者：「直接生就好」
    ↓
LLM: (silent tools)
  ✅ get_session 檢查 session 狀態
  ✅ generate_ta_options → 拿到 4-6 個 TA 候選
  ✅ validate_images（如果 session 有產品圖）+ digitize_product_text
  ✅ Infer pass（CLAUDE.md #6.5）：aspect/color/font 從對話 + brand-scrape signal 推；language 明確問；cta silent default；qa/testimonials 不列（post-gen）
    ↓
互動 1（使用者看 TA 候選、挑組）：
  「AI 切了 4 組 TA：[列 4 個]。你要挑 1 組生、還是挑 2 組各生一份？」

互動 2（Cost 複誦 + 推斷值一起給使用者反對機會）：
  「依你講的、我把規格都填好了：
   - 受眾：[使用者剛挑的] ← 你答的
   - 頁數：10 頁 ← 我幫你配（一般品牌首發、你沒指定）
   - 長寬比：9:16 ← 我幫你配（你前面提過 IG）
   - 語言：繁中 ← 我幫你配（你官網是繁中）
   - 色系：墨綠 #2fa067 ← 我幫你配（官網抓到的）
   - 字體：襯線 ← 我幫你配（你做保健食品、襯線合調性）
   - CTA：官網 ajoy.com.tw ← 我幫你配
   - Q&A：加 ← 我幫你配、猶豫型客人多
   - 見證：不加 ← 我幫你配、你沒給評價

   扣 2,000 pts（你餘額 123,610、夠用）。
   有標「我幫你配」的、特別看一下、要改哪項講哪項、
   全部 OK 就回「開始」我直接跑。」

互動 3（若使用者要改某項、就改；若全 OK 進 互動 4）

互動 4（使用者回「開始」、LLM pre-flight self-audit、call generate_session）
```

**關鍵差異**：
- 使用者答了 1-2 題（挑 TA + 確認規格），**不是 0 題**
- LLM 的推斷值**顯性列出、標「我幫你配」、附來源**
- Backend 工具**照樣 call**（generate_ta_options、validate_images 都跑）、只是使用者不被追問細節
- Cost 複誦**完整列 9 個欄位**、使用者可挑任一項反對

#### 「直接生」絕對不可以做的事

1. ❌ 不 call `generate_ta_options`、自己捏一個 TA 名稱就生
2. ❌ 不 call `validate_images`（使用者有產品圖時）
3. ❌ 推斷值不列出來、直接放進 Cost 複誦（使用者不知道自己同意了什麼）
4. ❌ 頁數自己定（8-21 之間、使用者沒講就**問一次**、不要預設 10）
5. ❌ 把 2 個以上的 TA 合成一個 persona（TA 是 per-TA 欄位、不同 TA 可不同設定）
6. ❌ 跳過 Cost 複誦、直接 `generate_session`

#### 只有一種情境可以進一步簡化（真正的「default run」）

使用者明確說過「我真的不在乎細節、AI 全配」**且**使用者沒給任何 signal（沒 URL、沒 brand、saleskit 諮詢也沒講清楚）→ LLM 還是要：
- ✅ 照跑 `generate_ta_options` + 展示候選（使用者可能看到 TA 才有感）
- ✅ 問頁數（這決定扣點、不能猜）
- ✅ 問 TA 組數（這也決定扣點）
- ✅ Cost 複誦所有 LLM 推斷值（即使使用者前面說「不在乎」、也要給他最後一次反對機會）

**頁數 + TA 數量是使用者唯一必須親口答的兩題**、不管他多不想回答。

---

## Phase 3: Trigger Generation

### Step 0 (MANDATORY): Pre-Flight Self-Audit — 呼叫 generate_session **之前**最後一關

**流程**：呼叫 `get_session` 把當前狀態撈回來、逐項對照 checklist、缺一項就回去補、不准用「差不多就好」精神蒙混。這是 session 扣點前最後一道自己把關的審查、**沒 backend 硬擋、所以你自己必須誠實跑**。

```
session = mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token, "session_id": session_id
})
shared = session["wizard_shared_data"] or {}
tas    = session["wizard_ta_groups"] or []
```

#### Self-Audit Checklist — 逐項勾才能往下

**Session 級（Shared fields）**：
- [ ] `shared["brand_name"]` 非空 — brand-onboard Step 2 有跑
- [ ] `shared["industry_category"]` 非空、且在合法 enum 裡（不是 Gemini 爬出來的中文字）
- [ ] `shared["aspect_ratio"]` 有明確值（9:16 / 16:9 / 1:1 / 4:5）
- [ ] `shared["cta_url"]` 有 URL（silent default 用 brand 官網、永遠有值、不會缺）
- [ ] `shared["cta_text"]` 有值（產業預設、永遠有值、不會缺）
- [ ] `shared["spokesperson_choice"]` 存在（Phase 3.5 必跑、跳關會 silent fallback）
- [ ] `shared["requested_stripe_count"]` 在 8-21 範圍內
<!-- 不檢查 include_qa_section / include_testimonials — phantom fields、backend 不消費 -->

**每個被選中的 TA（Per-TA fields）**：
對 `request.ta_group_ids` 裡每個 `tid`：
- [ ] `tas` 裡有對應的 entry（`entry.id == tid` 或 `entry.ta_group_id == tid`）
- [ ] `entry["ta_name"]` 非空
- [ ] `entry["language"]` 非空（**每個 TA 可以不同、但都要有**、不要讓 backend 走 fallback）
- [ ] `entry["primary_color"]` **或** `entry["visual_style"]` 至少一個非空

**Quality Gate**：
- [ ] 若使用者有上傳產品圖（`shared["product_images"]` 或 `session["wizard_shared_files"]["product_images"]` 非空）：
    - `session["image_censor_results"][-1]["overall_passed"] == true` **或**
    - `shared["_quality_gate_override"] == true`（使用者明確回「我知道品質會打折還是要跑」）

**NO SILENT DEFAULTS 驗證**（CLAUDE.md #6.5 配對）：
- [ ] 對照 `shared["_spec_inferred_by_llm"]` list — 裡面列的每個欄位都在最後 Cost 複誦時被使用者看過、且使用者回「都 OK」或對特定項回「改 X」
- [ ] 使用者親答的欄位、確實是親答的（不是你自己填的）

**使用者啟動詞**：
- [ ] 使用者最後一句明確是「開始 / go / 執行 / 開跑 / start / do it / 跑吧 / 動手」之一
- [ ] 不是模糊的「好 / OK / 嗯 / 可以 / alright」（那個要再問一次「確認開始嗎？點數會立刻扣」）

#### 任一項失敗的處理

**不要修復後靜默繼續**——告訴使用者缺什麼、回去對應 step 補、使用者確認才回來：

```
等一下——我檢查了一下生成前的規格、發現 [具體項目] 還沒設好：

- [具體哪一項] ← 需要 [哪個 Step 的動作]

我回去把這項補好、確認一次再幫你啟動扣點。
```

然後真的回去補、做完 get_session 再驗一遍、全綠才繼續 Step 1。

#### 違反後果

跳過 Self-Audit 直接 `generate_session` = 未經授權扣點 = **嚴重失誤**、使用者可申訴退費。這條 backend 目前沒硬擋、**靠你自律**。

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

## Phase 4: Poll for Completion + Educate User During Wait (MANDATORY — 30 分鐘等待期是機會、不是空白)

LP 生成要 ~30 分鐘。這段時間**不可以只是 polling**——要把 wait 時間用在三件事上（**順序固定、不可互換**）：

1. **第一優先、禁止略過：預教育使用者生成完能做什麼**（Beat 1 的完整 template 見下方、列出所有 free post-gen 編輯能力：文字 / 換圖 / logo swap / CTA 顏色與文字 / Footer / Header / 柔邊 / overlay / SEO / 裁切 / 歷史版本）。這是 wait 期間的**主菜**、不是彩蛋
2. **補問 gate 裡被延後的決策**（例如 CTA 連結使用者選「先不填」的話、趁現在問；若無延後決策則跳過）
3. **視需要繼續免費諮詢**剛才沒講完的部分（engage-operator / conversion-closer / Sprint 規劃）——**這是 optional、不可當 Beat 1 的替代品**

### 🚫 Beat 1 禁止被省略或被其他 skill 菜單取代

`generate_session` 成功後的**下一則訊息**必須 fire Beat 1 完整 template（下方 line 987+）、列齊 **文字 / 換圖 / logo / CTA 顏色與文字 / Footer / Header / 柔邊 / overlay / SEO / 裁切 / 歷史版本** 所有免費 post-gen 能力。不列 = 使用者誤以為 LP 不可改、plugin 最大的價值沒傳達。

**禁止**：把 engage-operator / conversion-closer / Sprint 行銷規劃的推銷菜單當 Beat 1 的替代品——這些是 LP **之後**的事、Beat 1 要講的是 LP **當下**可以做什麼。前者只能作為 Beat 1 後的 optional 補充。

**自檢**：觸發生成後的第一則訊息有沒有列出 edit-landing 的具體編輯動作（改 CTA 顏色 / 換 logo / 加柔邊 / 跑 SEO 這種動詞 + 具體 target）？沒有 = 違規重寫。

### Polling cadence

LP 生成約 30 分鐘，poll **每 60-90 秒一次**（不是每 5 秒——那是廣告用的節奏，LP 會浪費配額且使用者看到刷太快反而焦慮）。

```
mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token,
  "session_id": session_id
})
→ { "status": "generating" | "complete" | "error", ... }
```

### Three engagement beats during wait

#### Beat 1 — 觸發生成後「立刻」講一次（不是等第一次 poll 回來）

使用者語言對應的版本（繁中示例；英文 / 日文 / 韓文等自行翻譯，**保留三段結構**）：

```
LP 已經啟動生成，預計 30 分鐘左右。這段時間我先跟你講**等一下完成後你能做什麼**，我們邊等邊規劃。

🔗 **你會拿到的連結（3 個）**
1. **銷售頁預覽**：可以在手機 / 桌機直接打開看實際長相，手機版特別重要（多數客人從 IG 點進來）
2. **圖像編輯器**（拖拉介面）：不用會寫程式，可以直接在瀏覽器裡改文字、換圖、調版面
3. **全頁截圖（長圖）**：一張完整 LP 長圖，可以貼到 Line 群組、寄 email、印成 DM

✏️ **你可以做的編輯**（除了重生圖片 100 pts/張，其他都免費、記得列**所有**這些項目、不要只挑 2-3 個）：

**📝 內容類**
- **改文字 / 換圖 / 調版面**：對話裡講「把第 3 頁標題改成 XX」、「英雄圖換成我附的這張」
- **截圖圈選**：把 LP 用手機開起來、截圖、用 Mark 功能圈要改的地方貼給我——比打字講清楚
- **Q&A 區塊 / 客戶見證**：如果要加 FAQ 或使用者評價區塊、生成後再跟我講要加什麼內容

**🎨 視覺類**
- **CTA 按鈕**：文字（例「立即訂位 → 馬上預約」）、連結網址、顏色、圓角、字體大小都可以改
- **裁切 stripe**：每張 LP 圖片的邊界可以單獨調（裁掉不要的區域、調鏡頭重心）
- **柔邊**（頁與頁之間的漸層融合）：預設硬切、可以加 0-100% 柔邊效果讓轉場順
- **Overlay**：如果某張 stripe 上的文字被背景蓋到看不清、可以加半透明深/淺色層提升閱讀性

**🏷️ 品牌識別**
- **左上角 Header logo**：隨時可以換、上傳新圖就好
- **Header 導航列**：可以新增 / 刪除 / 重排上方的按鈕（例「關於我們 / 菜單 / 訂位」）
- **Footer**：底部文字、連結、版權資訊都能改
- **Header / Footer 顏色**：可以跟 stripe 主色分開設（例 stripe 暖金 + Footer 深黑）

**🔍 行銷類**
- **SEO / GEO / AEO 優化**（一鍵 AI 全自動，500 pts，beta 期免費）：Google + **AI 搜尋引擎**（ChatGPT / Perplexity / Claude 等 LLM 流量）的 meta tags / 結構化資料 / FAQ / `llms.txt` / GEO summary 全包

**🕐 版本管理**
- **歷史版本**：每次編輯都存版本、不滿意可以回到任一個舊版本（不是 undo/redo、是 version list）

🚀 **生成完後有 3 種上線方式**
1. **用 landingai.info 現成網址**（預設、最快）：拿到類似 `https://landingai.info/zh-TW/lp/...` 的網址就能直接發給客人，不用自己架站
2. **嵌入你自己的官網**：我給你 HTML 片段 + landing page config JSON，你放到自己網站對應的頁面裡
3. **一鍵發佈 / 投放廣告**：發 IG / FB / TikTok 貼文，或直接建 Meta / Google 廣告

等一下生成完後，你想走哪一條我都可以幫你走完。

---

**趁現在在跑，想確認 3 件事**（有助於生成完立刻上線）：
1. **CTA 按鈕要連到哪**？之前的規格單如果你選「先不填之後再加」，這一次生成出來的 LP 會是 placeholder——要不要趁現在給我網址？（官網 / LINE / 購買頁 / 預約頁）
2. **部署偏好**：想用 landingai.info 的網址（最快，立刻可用），還是要嵌進自己的官網（我會在完成後生 HTML + JSON 給你）？
3. **SEO**：要不要等一下生成完、我一鍵幫你做 SEO + GEO + AEO 優化？AI 會根據你的品牌自動生 meta tags + 結構化資料 + FAQ + `llms.txt` + GEO summary，**不用你給關鍵字**（但如果你有特定關鍵字想強打也可以提）
```

#### Beat 2 — 進度過半時（約 15 分鐘後）

**不要重複 Beat 1 的菜單**，只給「進度 + 一個具體補問」：

```
進度過半（還要約 15 分鐘）。
剛才 SEO 關鍵字你還沒回我，要不要現在想一下？生成完可以立刻套用、不用等另一輪。
```

或如果 CTA / 部署也還沒定：

```
進度過半。剛才 CTA 連結你還沒決定，最常見是先放 LINE 官方帳號——
要不要先用 LINE，之後再換？
```

#### Beat 3 — 快完成時（約 25-28 分鐘後）

```
快好了，剩幾分鐘。等一下會直接給你 3 個連結：
📱 銷售頁預覽 / ✏️ 拖拉編輯器 / 🖼️ 全頁長圖。
你可以先想一下第一眼會想看哪一頁——「hero 首屏」還是「CTA 結尾」最常先被檢視。
```

### Stage name translation — 遵守 JARGON BLACKLIST #12

顯示進度時，**翻成人話**，不准打英文階段名（即使加括號也不行）：

| Backend status | 使用者看到 |
|---|---|
| `strategizing` / `strategist_running` | 第 1/4 階段：分析品牌與受眾 |
| `architecting` / `architect_running` | 第 2/4 階段：設計版面與文案 |
| `factorying` / `factory_running` | 第 3/4 階段：生成視覺素材 |
| `reflecting` / `reflector_running` | 第 4/4 階段：品質檢查 |
| `completed` | 完成 |
| `failed` / `error` | 「遇到問題、我幫你重試一次」（不貼錯誤訊息） |

❌ 絕對禁止：「第 3 階段 工廠生成中（producing）」、「策略分析中（strategizing）」——**英文括號也算違規**。
✅ 正確：「第 3/4 階段：生成視覺素材」。

### Error Handling

- **"error" status**: 用人話講「遇到問題、我重試一次」；內部重試失敗 3 次以上才向使用者報告，並用中文描述卡點，不貼原始 error message
- **Timeout（30 次 poll ≈ 45 分鐘還沒完成）**: 「生成比預期久一點，你想再等還是等下次再看？」
- **Partial generation**: 「有 N 張沒生好，我幫你個別重生，每張 100 pts」

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
A) Edit text only (free, no regen) → tell me the stripe + new headline/subheadline/body, I'll update instantly
B) Edit visuals or layout → regenerate the affected stripe (100 pts/regen, free allowance = page count)
C) Screenshot editing → open the sales page, take a screenshot, circle what to change, paste here (I'll red-box annotate then regen)
D) Crop stripe image → `crop_stripe` tool, adjust the visible rectangle (free, no regen)
E) Soft-edge adjustment → `set_stripe_soft_edge` for smooth gradient edges between stripes (free)
F) Download individual stripes → `download_stripe` for one, `download_all_stripes` for full zip
G) Export HTML / full static image → `export_html` (full single-page HTML), stitched image from `export_landing_image`
H) SEO optimization → `seo_preflight` to check, then `run_seo_optimize` (500 pts) to generate meta tags, schema markup, keywords
I) Build homepage → embed this LP into a full website (/salecraft-homepage)
J) Publish → push to social media or run ads (/salecraft-publish)
K) Self-host the LP on your own domain → download stripes + LP config JSON, embed as static HTML block on your own site (see Phase 7 below)
L) Generate another variant (different TA or aspect ratio)

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
- The **sales page link** (landingai.info) is the PRIMARY link to show — this is the actual rendered page (landingai.info, not salecraft.ai — salecraft.ai is the marketing site, NOT the LP renderer)
- The **visual editor link** is the SECONDARY link for self-service editing
- The stitched_image_url is a tertiary static preview
- If multiple LPs were generated (different TAs), provide a sales page link for EACH campaign_id
- The locale in the URL should match the user's language (zh-TW, en, ja, etc.)
- ALWAYS include the visual editor link — many users prefer drag-and-drop

---

## Phase 7: Self-Host Deploy Guidance (只在使用者問「能不能放到自己官網」才主動說明)

使用者若說「我想把這個 LP 嵌到自己的網站」「放到我自己的 domain」「不想用 landingai.info 的網址」——那才走這個流程。**不要主動推**、使用者多半用 salecraft.ai 託管版就夠了。

### 兩種佈署模式

**A. 輕量嵌入（只要圖）**：
```
# 下載所有 stripe 圖
mcp_tool_call("landing_ai_mcp", "download_all_stripes", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ 回傳 zip 連結，含每 stripe 的 WebP / PNG
```
使用者把 zip 下載到自己機器、再上傳到自己網站 static assets。自己寫 HTML `<img>` 堆疊即可、LP 的 interactivity（scroll / CTA hover）要自己補。

**B. 完整帶 config（保留動線互動）**：
```
# 匯出 HTML + config
mcp_tool_call("landing_ai_mcp", "export_html", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ 回傳完整 standalone HTML（含 stripe 圖 URL + CTA button + font + 所有 stripe 結構）
```
把回傳的 HTML 放進自己網站的 `<iframe>` 或直接用 `<div>` 嵌入。LP config JSON 保留 stripe order / cta_text / primary_color 等，之後要改可回 salecraft.ai 編輯再重新 export。

### 需要提醒使用者的事

- ❗ **圖 URL 是 GCS host 的**（`storage.googleapis.com/...`）。自己網站直接引用會繼續從 GCS 拉圖、**不佔你網站流量**、圖跟著 LP 一起 refresh
- ❗ 若要完全斷開 salecraft 依賴（自己 host 圖），得把 GCS URL 的圖全部下載下來、改成自己網站的 `<img src>`，然後 CTA button 的 click tracking 會失效（salecraft.ai 的追蹤 pixel 不在）
- ❗ LP 在 salecraft.ai 編輯、export 一次 → 放到自己站 → 再改時要重新 export 一次貼回去。沒有自動同步

---

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

**🧠 Auto-record to memory (silent):**
```
POST /ai-agent/brand-memory/save-prompt
{ "brand_id": "<brand_id>", "product_name": "<product>",
  "prompt_type": "generation", "user_input": "<what user asked to generate>",
  "ai_response_summary": "Generated LP with <N> pages, campaign_id=<id>, cost <N> pts",
  "skill_used": "generate-landing", "resulted_in_paid": true,
  "satisfaction_signal": "positive" }
```
Also compile metadata: `POST /ai-agent/brand-memory/compile-metadata { "brand_id": "<brand_id>" }`

**Then show the user — 這個 menu 是 MANDATORY、不准省略不准只講 3 項**：

LLM 常犯：生完只回「好了、你看看要不要發佈 / 做 reels」三選一、使用者不知道還能做柔邊 / overlay / 單頁重生 / SEO 一鍵 / 調 CTA 等細部設定。**必須主動把下面這整份選單展開給使用者看**、讓他知道選項全貌。

```
✅ LP 生成完畢！

🔗 你的連結
1. 📱 銷售頁預覽：[sales_page_url]
2. ✏️ 圖像編輯器：[editor_url]（網頁視覺編輯、拖拉改文字 / 換圖）

---

📋 接下來可以做的事（分三類）

**🎨 細修（不用扣點的視覺 / 文字微調）**

對使用者介紹每個功能時、**都要附一句比喻讓他聽懂**（不要只講工具名）：

- ✏️ **改文字** — 某一頁文案、標題、副標、CTA 按鈕字（`update_stripe_text / update_stripe_texts`、**免費**）
- 🔗 **CTA 按鈕連結** — 按鈕連到哪（官網 / LINE / 購買頁 / 預約 / App 下載 / 滑到某區塊、`update_cta / update_cta_link`、**免費**）
- 🎨 **改 CTA 按鈕樣式** — 顏色 / 圓角 / 陰影（`update_cta_style`、**免費**）
- 🌊 **加柔邊**（`set_stripe_soft_edge`、**免費**）— 比喻：「頁跟頁的**交界**不要硬切、像水彩畫邊緣**暈染**過去、整頁往下滑更順」。參數 `top` / `bottom`（0.0-0.3 建議）：**數字越大 = 柔邊越強 / 邊緣越模糊融合**、0 = 完全銳利硬切。使用者若講「100%」要**反問**：「是想要柔邊**最強**還是**完全關掉**？兩個都有人這樣叫。」
- 🎭 **加深淺 overlay**（`set_stripe_overlay`、**免費**）— 比喻：「像在圖片上放一層**半透明的有色玻璃**、讓亮背景變暗、暗背景變亮、白字 / 黑字才讀得清楚」。參數 `color`（`#000000` 深、`#FFFFFF` 淺）+ `opacity`（0.3-0.5 典型）
- ✂️ **裁切某頁**（`crop_stripe` / `reset_crop`、**免費**）— 比喻：「像相機**變焦放大**、只改你看到的範圍、不重新拍」。使用者講模糊位置（「切到大拇指那邊」）時、**先反問**「保留上方多少 %」或「上 / 中 / 下哪一塊」、再算成 normalized 座標。詳細流程見 `edit-landing` 的 Crop a stripe 段
- 🖼️ **改背景** — 換色 / 換圖（`update_stripe_background`、**免費**）
- 🧭 **Header 按鈕管理** — LP 最上方那排按鈕：新增 / 刪除 / 調順序 / 改連結（`patch_landing_config` 整個 `header_nav` 陣列覆寫；不確定要放什麼時 LLM 自己依 LP 內容判斷給建議、**免費**）
- 📸 **截圖標註** — 你圈紅框標哪裡要改、我直接動（**免費**）

**🔧 結構 / 重生（扣點）**
- 🔁 **單頁重生** — 某頁不滿意、用新 prompt 重跑（100 pts / 頁、`regenerate_stripe`、**舊版自動留著**）
- 🕒 **歷史版本** — 同一頁生過多次？列出所有版本、挑哪版都可以切回去（`get_stripe_history` + switch、免費）
- 🙈 **隱藏 / 顯示某頁** — 暫時藏起來不刪掉（`hide_stripe` / `restore_stripe`）

**🚀 上線 / 分享（大部分免費）**
- 🔍 **SEO 一鍵優化** — AI 全自動生 meta / Open Graph / JSON-LD / FAQ schema / llms.txt / GEO summary（500 pts、beta 期免費、`run_seo_optimize`）
- 🏠 **建立首頁** — 把這個 LP 嵌進完整首頁（`homepage-builder`）
- 📤 **發到這個 LP 到社群** — IG / FB / TikTok / Threads（`publish-social`）
- 📊 **拿這個 LP 投廣告** — Meta / Google / LinkedIn 建廣告活動（`publish-ads`）
- 🔁 **生另一個版本 / 其他語言** — 同 brand 換 TA / 換語言（英文、日文…）/ 換 aspect_ratio / 換風格再跑一次（**新 session、重新扣點**）。⚠️ plugin 沒有「便宜翻譯」路徑、想要其他語言就是重生一份完整 LP、不是翻譯既有的

**不屬於這個選單的**（是獨立產品、不是「對這個 LP 的操作」）：
- 🎬 短影音 Reels → `/salecraft-reels` 獨立流程、不是 LP post-gen 動作
- 其他新品牌 / 新產品相關生成也一樣、不混入 LP 選單

告訴我你想先做哪個、或直接講「我要改 XX」、「幫我 SEO」、「發到 IG」，我就接手。也可以先點上面連結看成品、想到要改什麼再說。
```

### CTA button reminder (若 cta_url 使用者沒明確填過)

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
