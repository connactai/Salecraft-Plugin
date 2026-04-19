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

## Phase 2.9: Pre-Generation Confirmation Gate (MANDATORY — 不可跳過)

**⚠️ 這是這個 skill 最容易失敗的一關。** 上面 Phase 1-2.8 都只是準備，**真正扣點的是 Phase 3 的 `generate_session`**。一旦呼叫 `generate_session`，使用者的點數就扣了，LP 規格也鎖死——所以在這之前必須走完 **12 個 HARD STOP GATES**（定義在 CLAUDE.md → "HARD STOP GATES — 啟動付費生成前必問的 12 個確認點"）。

### 13 Gate Checklist（全部勾完才能進 Phase 3）

- [ ] 1. 素材來源（URL / Google Drive / 手動 / 都沒有——4 選項必列）
- [ ] 2. Logo 再三確認（有沒有？沒提供的話 AI 會隨便配）
- [ ] 3. 產品圖再三確認（有產品實拍嗎？沒有的話 AI 會從文字瞎想產品樣子，截圖給客戶就露餡）
- [ ] 4. 代言人再三確認（**獨立一題、不可跟 Gate 3 合問**）— 必列三選一：
    - 📸 **上傳自己的照片**（個人品牌 / 履歷適用）→ 走 `brand-onboard` Phase 3.5 上傳流程、`create_spokesperson(is_ai_generated=false, photo_urls_json=[...])`
    - 🤖 **AI 自動生成代言人**（AI 會生一張擬真人臉、看起來很真但不是真人——不能拿去做對外代言、避免法律風險）→ **必須另外問 9 題參數**：性別 / 年齡 / 族裔 / 眼鏡 / 體態 / 穿著 / 氣質 / 髮型 / 加分細節。詳見 `brand-onboard` Phase 3.5 → "AI-Generated Spokesperson — Parameter Collection"。**絕對禁止略過這 9 題直接 `is_ai_generated=true`**，否則使用者拿到的人不會是他腦海中的樣子
    - 🚫 **不使用人物**（純文字 + 圖形 + 產品圖）→ 跳過代言人、不 `create_spokesperson`
  **絕對禁止因使用者有產品圖就假設代言人也處理完** — 這兩件事是獨立 gate
- [ ] 5. TA 數量（每組多 1,600 pts 寫清楚）
- [ ] 6. 長寬比（橫版 / 直版 / 兩者都要）
- [ ] 7. 頁數（**8-21 頁範圍**，200 pts/頁；依使用者內容量推薦，不要上來就給 8/10 二選一）
- [ ] 8. 語言（15 個選項）
- [ ] 9. 色系（品牌色 / 情緒詞 / 交給 AI）
- [ ] 10. 字體風格（手寫 / 襯線 / 無襯線 / 交給 AI）
- [ ] 11. CTA 按鈕連結（官網 / 購買頁 / LINE / 預約 / 先不填）
- [ ] 12. 是否加 Q&A / 常見問題區塊
- [ ] 13. 是否加客戶見證 / 評價區塊

**問法**：分 3 批（素材組 1-4 / 規格組 5-8 / 風格組 9-13），每批 3-4 題，維持對話節奏。**不要一次把 13 題全丟出來**。

### 🔴 MANDATORY: 每批答完就 `update_session` 寫進 backend（不要只在對話裡記）

**痛點**：LLM 問完 gates 只在對話 context 記答案、沒寫回 session。若對話太長 context 被裁掉、使用者換 AI、session 被另一個 AI 接手 → 答案全失。使用者以為「已登記」結果 backend session 還是空的。**這會在 Phase 3 cost 複誦時露餡，或更糟是 generate_session 用預設值導致使用者預期跟實際 LP 不符**。

**強制規則**：使用者答完一批（3-4 題）後，**立刻**呼叫 `update_session` 把那批答案寫進 `wizard_shared_data`（共用）或對應的 TA 條目（`wizard_ta_groups`），再進下一批。

#### 批 1（素材組 1-4）寫入範例

素材組很多答案是在 `brand-onboard` 早就寫進去了（logo / 產品圖 / 代言人 / 認證圖桶）。這批的 update 通常只是補 Gate 4 代言人選擇結果：

```
# 使用者選了「AI 生成代言人」+ 9 題參數
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_shared_data\": {\"spokesperson_choice\": \"ai_generated\", \"spokesperson_params\": {...9 params...}}}"
})
# 若選「不使用人物」則寫 "spokesperson_choice": "none"
# 若選「上傳自己照片」則 create_spokesperson 已處理、不需再 update_session
```

#### 批 2（規格組 5-8）寫入範例

```
# 答完 TA / 長寬比 / 頁數 / 語言
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{
    \"wizard_shared_data\": {
      \"aspect_ratio\": \"9:16\",
      \"requested_stripe_count\": 10,
      \"language\": \"zh-TW\"
    },
    \"wizard_ta_groups\": [<被選中的 TA objects — 同 Phase 2.5 格式>]
  }"
})
# 說明：頁數 = page_count = requested_stripe_count（之後 generate_session 會吃這個）；
#       aspect_ratio 允許值 9:16 / 16:9 / 1:1 / 4:5；
#       language 用 ISO code（en / zh-TW / ja / ...），backend 會透過 _normalize_language 轉 prompt-literal。
```

#### 批 3（風格組 9-13）寫入範例

```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_shared_data\": {
    \"primary_color\": \"#2fa067\",                   # 或 \"warm_green_healing\" 情緒詞、或 \"auto\" 交給 AI
    \"font_style\": \"handwritten|serif|sans_serif|auto\",
    \"cta_url\": \"https://line.me/R/...\",
    \"cta_text\": \"立即預約\",
    \"include_qa_section\": true,
    \"include_testimonials\": false
  }}"
})
```

#### 批寫入完成後驗證

每批 update 完、**一定要 `get_session` 讀一次**，檢查剛才寫的欄位真的有進 session：

```
session_state = mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token, "session_id": session_id
})
# 確認 wizard_shared_data 裡有 aspect_ratio / language / primary_color / ... 該有的欄位
# 若缺任何一項 → 那批 update 實際沒 persist、要重新寫
```

這個讀回驗證步驟不可省。使用者「已登記」的感覺必須是 backend 真的有寫。

### 最後 Cost 複誦 + 啟動確認（強制範本）

13 題問完、每批 `update_session` 寫進去、也每批 `get_session` 讀回驗證過之後，**再做一次 `get_session` 把所有欄位撈出來當複誦基準**——**不准用你對話記憶裡的版本**。

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

```
好，幫你整理一下（以下都是你告訴我的、我寫進系統的）：
- 受眾：[tas 裡每個 ta_name 列出]
- 頁數：[shared.requested_stripe_count] 頁 × [len(tas)] 組 = [total_stripes] 頁
- 長寬比：[shared.aspect_ratio]
- 語言：[shared.language]
- 色系：[shared.primary_color 或 shared.color_mood_keyword]
- 字體：[shared.font_style]
- CTA：[shared.cta_text] → [shared.cta_url]
- 附加：[shared.include_qa_section ✓/✗，shared.include_testimonials ✓/✗]
- 預計扣點：[stripe_cost × stripe_count × num_tas] pts（約 $[USD]）

你目前餘額 [X] pts。確認要開始嗎？
回「開始」我就跑；回「改 XX」就調整；回「取消」就先不動。
```

### 🔴 絕對禁止 — 複誦時不要列每頁內容

**反模式**（使用者回饋實測踩到）：

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

### 跳過 Gate 的唯一合法情境

使用者明確說「我不想回答這麼多問題，直接用預設跑」時：
1. **明確警告**：「OK，那系統會自己配色、字體、logo 樣子——之後不滿意要重生，每頁 100 pts。確定用預設跑？」
2. 等他再說一次 YES 才進 Phase 3
3. 即使走預設，**仍要確認頁數（8-21 之間）和 TA 數量**——這兩項直接決定扣點，不能幫使用者決定

---

## Phase 3: Trigger Generation

### Step 0 (NEW): Verify Phase 2.9 Gate passed

**DO NOT call `generate_session` if any of these are true:**
- 12 gates 沒全問完
- 使用者沒給明確啟動詞（只回「好」「OK」不算）
- 成本複誦沒做

違反這條 = 未經授權扣點 = **嚴重失誤**，使用者可申訴退費。

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

LP 生成要 ~30 分鐘。這段時間**不可以只是 polling**——要把 wait 時間用在三件事上：
1. **預教育**使用者生成完**能做什麼**（檢視 / 編輯 / 部署 / 發佈）
2. **補問 gate 裡被延後的決策**（例如 CTA 連結使用者選「先不填」的話，趁現在問）
3. **繼續免費諮詢**剛才沒講完的部分（互動策略、成交腳本、會員經營）

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

✏️ **你可以做的編輯**（除了重生圖片 100 pts/張，其他都免費）
- **改文字 / 換圖 / 調版面**：對話裡跟我講要改什麼，例如「把第 3 頁標題改成 XX」、「英雄圖換成我附的這張」
- **截圖圈選**：把 LP 用手機開起來、截圖、用 Mark 功能圈起要改的地方貼給我——**比打字講清楚**，尤其改位置類的動作
- **裁切 / 柔邊**：每張圖的邊界可以單獨裁、邊緣淡出效果可以加
- **SEO / GEO / AEO 優化**（一鍵 AI 全自動，500 pts，beta 期免費）：Google 搜尋 + **AI 搜尋引擎**（ChatGPT / Perplexity / Claude 等 LLM 也會成為流量來源）的 meta tags / 結構化資料 / FAQ / `llms.txt` / GEO summary 全部一起做
- **換 header logo**：如果左上角 logo 之後想換、告訴我，上傳新圖就好（不扣點）

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

**Then show the user:**
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
