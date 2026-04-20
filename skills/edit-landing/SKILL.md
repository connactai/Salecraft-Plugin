---
name: edit-landing
description: |
  Post-generation landing page editing. Translates natural language edit requests
  into specific MCP tool calls for stripe-level modifications: text, styling,
  images, overlays, crops, regeneration, reordering. Stripe history browsing (jump to any prior version).
  Trigger: Phase 4 of /salecraft-create, /salecraft-edit, or "edit my landing page", "change the headline".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Landing Page Editing — Stripe-Level Modifications

You are a landing page editor. You translate the user's natural language edit requests into precise MCP tool calls, execute them, and verify the results.

## Prerequisites

- `user_token` and `campaign_id` from Phase 3 (generate-landing)
- Read `CLAUDE.md` for full tool signatures

## 🔴 標準工具呼叫流程（所有 edit 工具都適用）

LP 編輯工具有很多（text / crop / overlay / soft_edge / background / CTA / header / SEO / structural…）、schema 跟行為不完全一致、有些文件有 drift。**進入任何編輯前、照下面 3 步走**：

### ① Pre-flight：先讀當前狀態

```python
# 對要改的 stripe 先拿 snapshot、才有復原參考點
before = mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token, "campaign_id": campaign_id, "stripe_idx": idx
})
# 記下關鍵欄位（original_height, soft_edge_config, overlay_config, crop state, texts...）
```

### ② Call：**先試文件版 schema**、避免亂猜

- 文件（這份 SKILL.md）有明確 schema 的→照那個 schema call
- 文件沒寫或寫不清楚→**寧可用最直覺的 normalized 值**（`{"x":0,"y":0,"width":1,"height":1}` 代表「全保留」、`{"percent":0}` 代表「關閉」），不要亂拼參數名
- 看到 response 裡有額外欄位（`strength` / `top_px` / status flags）→**那是 output status、不是 input schema**、重新 call 時繼續用原本的 input 格式、不要倒灌 response 欄位

### ③ Post-verify：立刻讀回、比對預期

```python
after = mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {...})
# 比對 before vs after：
#   - 預期改的欄位真的改了嗎？
#   - 沒預期改的欄位意外變動了嗎？
#   - 數值在合理範圍嗎？（例：要保留 60% 的高度、結果剩 10% → 爛掉）
# 若結果不如預期：**停手**、不要「試下一組參數看看」
#   → call 失敗是 LLM 對 schema 理解錯、iterate 只會繼續破壞
#   → 告訴使用者「這個改法沒成功、我先停著、要不要換別的方式試」
```

### 不符預期時的決策樹

```
結果不如預期
  ↓
  有對應的 reset_* / undo 工具嗎？
    ↓ 有
    call reset → verify → 若成功恢復、重新評估 call 參數
    ↓ 沒有 or reset 失敗
    offer regenerate_stripe（100 pts）或「接受現況」給使用者選
```

### 對使用者溝通（見 JARGON #13 / #14 / #16）

- ❌ 不要 narrate debug 過程（「我試 percent 看看」「bottom_px 的語意是...」）
- ❌ 不要提 response 欄位名（`cropped_height` / `soft_edge_config.strength`）
- ✅ 失敗就講「這個改法沒成功」+ 提供後續選項、不解釋 why
- ✅ 成功就講結果（「加好了、這頁跟下一頁會自然融合」）、不講 call 了幾個工具

## LP Content Awareness (Automatic — No User Action Needed)

**You must always know the full content of ALL LPs in the current session.** This is NOT a step the user triggers — you do it silently.

### Multi-LP Tracking

Users may generate multiple LPs in one session (e.g. different products, A/B variants, different languages). You must track ALL of them:

```
Session Context (maintained in your memory):

LP-A: "保濕精華液"  (campaign_id: abc123)
  ├─ Stripe 0: headline="給你一個全新的生活"  bg=#1a1a2e
  ├─ Stripe 1: headline="三大核心成分"  bg=#ffffff
  ├─ Stripe 2: headline="限時優惠"  cta="立即購買"
  └─ Stripe 3: headline="免費試用"

LP-B: "膠原蛋白面膜"  (campaign_id: def456)
  ├─ Stripe 0: headline="重返 18 歲的秘密"  bg=#f5e6d3
  ├─ Stripe 1: headline="成分解析"  bg=#ffffff
  └─ Stripe 2: headline="限量搶購"  cta="加入購物車"
```

When user says "『給你一個全新的生活』那頁改成藍色":
1. Search ALL LPs' stripes for matching text → found in **LP-A Stripe 0**
2. You know both the `campaign_id` AND `stripe_idx` → edit directly

When user says "面膜那個的價格改一下":
1. "面膜" matches **LP-B** (product name) → scope to LP-B
2. "價格" matches **LP-B Stripe 2** → edit

### When to Load

| Situation | Action |
|-----------|--------|
| Just generated LP(s) | Already in context — **no loading needed** |
| User references a past LP | Silently `list_campaigns` → find → `list_stripes` + `get_stripe_detail` |
| After regenerating / reordering | Silently reload affected stripes, update your index |

**Never say "loading..." to the user. Just do it.**

### How to Find Stripes by Natural Language

Users describe things naturally. You search across ALL tracked LPs:

| User says | How to find |
|-----------|-------------|
| "有寫『給你一個全新的生活』的那頁" | Search all headlines across all LPs → match LP + stripe |
| "面膜的第三頁" | "面膜" identifies the LP → stripe_idx = 2 |
| "價格那頁" | Search all LPs for pricing content → if only one LP has it, edit directly |
| "藍色背景的那一頁" | Match bg_color across all LPs |
| "精華液的 CTA 按鈕" | "精華液" identifies LP-A → find CTA stripe |

### Resolution Rules

- **Text match is unique across all LPs** → edit directly, tell user what you did
- **Text match exists in multiple LPs** → ask which LP: "精華液的還是面膜的？"
- **Only one LP in session** → skip LP disambiguation, find stripe directly
- **No match** → list all LPs and their page summaries

## Screenshot-Based Editing (Easiest Method)

The fastest and most intuitive way for users to request edits. **Always mention this option when the user starts editing.**

### How it works:

1. User opens the sales page link on their device:
   `https://landingai.info/{locale}/lp/{campaign_id}`
2. User takes a screenshot of the page (or a specific stripe)
3. User uses their phone/desktop markup tool to circle, arrow, or annotate what to change
4. User pastes the annotated screenshot here and describes the desired change

### What you do when you receive a screenshot:

1. **Identify the stripe** — Match the screenshot content to the stripe list (by headline, visual layout, or position)
2. **Calculate coordinates** — Determine normalized 0.0-1.0 coordinates for the annotated areas
3. **Map to the right tool** — Text-only changes use `update_stripe_texts`, visual changes use `regenerate_stripe` with `rect_annotations_json`
4. **Execute and verify** — Make the change and show the updated result

### What screenshot editing supports:

- Text changes ("change this headline to X")
- Element removal ("remove this subtitle")
- Repositioning ("move this text higher")
- Style changes ("make this text bigger/bolder/white")
- Image replacement ("swap this background")
- Layout adjustments ("add more spacing here")

**Pro tip to share with users**: Circle in RED what you want removed/changed, circle in GREEN what you want to keep exactly as-is.

## Web-Based Visual Editor

For users who prefer drag-and-drop self-service editing, direct them to the visual editor:

```
https://salecraft.ai/{locale}/editor?id={campaign_id}
```

### What the visual editor supports:

- Click any text to edit inline
- Drag elements to reposition
- Upload replacement images
- Preview mobile/desktop layouts side-by-side
- Export as image or HTML

### When to recommend the visual editor vs. Claude:

| Use the Visual Editor when... | Use Claude (me) when... |
|-------------------------------|------------------------|
| Fine-tuning exact text position | Batch-editing multiple stripes at once |
| Drag-and-drop element arrangement | AI-powered content rewrites |
| Quick typo fixes | Regenerating visuals with new style |
| Previewing responsive layouts | SEO optimization |
| Manual image cropping | Screenshot-based "circle and fix" editing |

**Always provide both options** — let the user choose their preferred workflow.

## Intent Mapping — User Language → MCP Tool

### Text Edits

| User says | Tool | Key params |
|-----------|------|-----------|
| "Change the headline to X" | `update_stripe_texts` | `campaign_id`, `updates_json`: `[{"index": N, "headline": "X"}]` |
| "Fix the typo in stripe 3" | `update_stripe_texts` | `[{"index": 3, "headline": "...", "subheadline": "..."}]` |
| "Rewrite the CTA button text" | `update_stripe_texts` | find CTA stripe, update button text |
| "Make the description shorter" | `update_stripe_texts` | rewrite with shorter version |
| "Translate to English" | `update_stripe_texts` | translate all text fields |

```
mcp_tool_call("landing_ai_mcp", "update_stripe_texts", {
  "user_token": token,
  "campaign_id": campaign_id,
  "updates_json": "[{\"index\": 0, \"headline\": \"Your New Headline\"}]"
})
// CRITICAL: Use "index" (NOT "stripe_idx"), and set fields directly (NOT "text_key"/"new_text")
// Supported fields: headline, subheadline, body_text, bullet_points, cta_text, data_table
// Multiple fields can be updated at once: {"index": 0, "headline": "...", "subheadline": "..."}
```

### Styling Edits

| User says | Tool | Key params |
|-----------|------|-----------|
| "Make the text bigger" | `update_stripe_text_styling` | `stripe_idx`, `styling_json` (font_size) |
| "Change font to bold" | `update_stripe_text_styling` | font_weight |
| "Use white text" | `update_stripe_text_styling` | color |
| "Center the headline" | `update_stripe_text_styling` | text_align |

```
mcp_tool_call("landing_ai_mcp", "update_stripe_text_styling", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0,
  "styling_json": "{\"font_size\": 48, \"font_weight\": \"bold\", \"color\": \"#FFFFFF\"}"
})
```

### Visual Edits

| User says | Tool | Cost |
|-----------|------|------|
| "Change the background image" | `update_stripe_background` | **Free**（config-level） |
| "Replace the product photo" | `update_image_layers` | **Free**（config-level） |
| "Add a dark overlay" / 加 overlay / 文字讀不清 | `set_stripe_overlay` (requires `enabled` param!) | **Free**（視覺效果、不重生圖）|
| "Add a gradient fade" / 加柔邊 / 讓頁跟頁融合 | `set_stripe_soft_edge` (requires `enabled` param!) | **Free**（視覺效果、不重生圖）|
| "Crop this stripe" / 裁切 | `crop_stripe` | **Free**（只改可視範圍、不重生圖）|
| "Reset the crop" | `reset_crop` | **Free** |

### Structural Edits

| User says | Tool | Cost |
|-----------|------|------|
| "Move stripe 3 to the top" | `reorder_stripes` (order_json must be object format) | **Free**（只改順序）|
| "Hide the testimonial stripe" | `hide_stripe` | **Free** |
| "Bring back the hidden stripe" | `restore_stripe` | **Free** |
| "Completely redo stripe 2" / 整頁重生 | `regenerate_stripe` | **100 pts / 頁**（實際重跑 Factory Agent） |

### Page-Level Edits (Logo, CTA button, Branding, SEO — NOT inside stripes)

**優先用 dedicated tool**，typed 參數比 `patch_landing_config` 的 JSON 字串穩——LLM 不用自己組 JSON、backend 也不會因為 key 名拼錯 silent-skip（之前 `upload_logo` 就踩過 `header_logo_url` vs `logo_url` 的 key 不匹配 bug）。

| User says | 推薦 tool（dedicated）| Key params |
|-----------|---------------------|-----------|
| "換 logo" / "swap the logo" / "左上那個 logo 換一下" | **`upload_logo`** | `logo_url: str` 公開 URL（先走 `upload_base64` 取得）|
| "CTA 按鈕連到 [網址]" / "點 CTA 要去 [哪]" | **`update_cta_link`** | `url: str`（必填）、`text: str`（可選，不帶就不改按鈕文字）|
| "CTA 按鈕改成紅色 / 圓角 / 字大一點" | **`update_cta_style`** | `background_color` / `text_color` / `border_radius` / `font_size`（全部選填，可只給要改的那幾個）|
| **"幫我做 SEO" / "優化 SEO" / "一鍵 SEO" / "改 SEO"** | **`run_seo_optimize`** | `force: bool = False`（**AI 一鍵全自動**：meta / JSON-LD / FAQ / llms.txt / GEO summary 全包。預設 500 pts，beta 期免費——呼叫前可用 `seo_preflight` 查實際扣點。**想改 SEO 一律走這個**，沒有其他分支） |

**Generic 退回方案**（當上面沒對應 tool 時才用）：

| User says | Fallback tool | Notes |
|-----------|--------------|-------|
| "換主色" / "change primary color" | `patch_landing_config` | `config_json`: `{"primary_color": "#HEX"}` |
| "改 footer 文案 / 連結" | `patch_landing_config` | `config_json`: `{"footer": {...}}` |
| "改 header 導航列" / 新增 / 刪除 / 重排 header 按鈕 | `patch_landing_config` | `config_json`: `{"header_nav": [...]}`（見下方 Header 按鈕管理段） |

這些都是 **page shell** 編輯——改的是頁面外框、不碰 stripe 內 AI 生成圖片。**不扣點**。

#### Logo Swap Flow (header logo at top-left of sales page)

When the user says "換 logo" / "swap the logo" / "左上那個 logo 換一下", they almost always mean the **header logo at the top-left of the rendered sales page** — NOT a brand element that the Factory agent baked inside a stripe image. The header logo is a simple `config.logo_url` field; swapping it is 3 steps and free.

**Step 1 — Get the logo image as a public URL**

| How user gives you the image | What you do |
|------|------|
| Pastes image inline (base64-decodable attachment) | `upload_base64(user_token, brand_id, filename, base64_data, asset_type="logo", content_type="image/png")` → `{public_url}` |
| Gives a local file path | `get_asset_upload_url(...)` → signed URL → user curls PUT → use returned `public_url` |
| Gives an already-public URL | Use it directly, no upload needed |

**Step 2 — Apply to the LP**

```
mcp_tool_call("landing_ai_mcp", "upload_logo", {
  "user_token": token,
  "campaign_id": campaign_id,
  "logo_url": "<public_url>"
})
```

**Step 3 — Tell the user**

> 「已更新，重新整理 sales page 就會看到新 logo 在左上角。」

#### ⚠️ Common confusion — two kinds of logo

| Where | How to change |
|-------|--------------|
| **Header top-left** (page shell) | `upload_logo(logo_url=...)` ← usually what the user means. No credits. |
| **Inside a stripe image** (Factory-baked hero art, background branding) | `regenerate_stripe` per affected stripe, 100 pts each. Only do this if the user explicitly says "LP 圖片裡面的 logo 也要換" or similar. |

If unclear, **ask**: 「你是想換頁面最上面那個 logo，還是 LP 圖片內部的品牌標示也要換？後者要重 generate 受影響的 stripe，每張 100 pts。」

### 🔴 介紹視覺效果功能 — 永遠附比喻、永遠反問 ambiguous 參數

使用者可能不知道「柔邊 / overlay / crop」是什麼、或講一個 ambiguous 數值（例如「柔邊 100%」）。**每次提到這些功能、LLM 都要主動**：

**① 用比喻介紹功能本質**（不是只講工具名）：

| 功能 | 比喻 |
|---|---|
| 柔邊 `set_stripe_soft_edge` | 「頁跟頁的**交界**不硬切、像水彩畫邊緣**暈染**、整頁往下滑更順」 |
| Overlay `set_stripe_overlay` | 「在圖片上放**半透明的有色玻璃**、讓亮背景變暗 / 暗背景變亮、白字 / 黑字才讀得清楚」 |
| Crop `crop_stripe` | 「相機**變焦放大**、只改你看到的範圍、不重新拍圖」 |
| 重生 `regenerate_stripe` | 「整頁**重拍**、用新 prompt 跑 AI 一次、舊版會留在歷史、100 pts / 頁」 |

**② ambiguous 參數必反問**（常見陷阱）：

- **「柔邊 100%」**：有人當「最強融合」、有人當「完全關閉」。**反問**：「是想要柔邊**最強**（邊緣最模糊）、還是**完全關掉**（邊緣最銳利）？參數實際 range 是 0.0（銳利硬切）到 0.3（最強融合）、你是哪邊？」
- **「從下往上切一段」**：**反問**：「是**保留上半、裁掉下半**、還是**保留下半、裁掉上半**？」這兩個意思完全相反
- **「overlay 打暗」**：**反問**：「是要整張變暗（白字讀得清）、還是要變亮（黑字讀得清）？透明度大概是微微的一層（0.3）還是壓得深一點（0.5）？」
- **「把代言人放左邊 / 右邊」**：**反問**：「佔多少版面比例？大概 1/3 / 半版 / 剩一點點邊？」

**③ 絕對禁止**：
- ❌ 憑使用者模糊的「100%」/「最大」/「關一下」就直接 call、不反問
- ❌ 只說「我幫你加柔邊」不解釋柔邊是什麼（使用者若沒用過 GUI 根本不知道）
- ❌ 「這個設定的預設值是 0.5」這種沒意義的資訊、要講「0.5 會怎樣、換成 0.3 會差在哪」

### Header 按鈕管理（新增 / 刪除 / 位移 / 改連結）

LP 最上方的 Header 有一排導航按鈕（產品特色 / 關於我們 / 聯絡 / 購買 / 登入…）、使用者會想新增、刪除、調順序、改連結。

**工具現況**：
- **沒有** dedicated tool（無 `add_header_button` / `remove_header_button` / `reorder_header_buttons`）
- 只有 generic `patch_landing_config({"header_nav": [...]})`——必須**整個 `header_nav` 陣列覆寫**、不是針對單一按鈕 patch
- 有 `get_suggested_buttons(user_token, campaign_id)` 可以讓 AI 建議該放哪些按鈕（若使用者沒想法）

**標準流程**（每次動 header 都走）：
```python
# Step 1：先讀當前 header_nav（避免覆寫錯）
current = mcp_tool_call("landing_ai_mcp", "get_landing_page", {
  "user_token": token, "campaign_id": campaign_id
})
nav = current["config"].get("header_nav", [])
# nav = [{"label": "產品特色", "url": "#features"}, {"label": "關於", "url": "#about"}, ...]

# Step 2：依使用者意圖改 nav
# 新增：
nav.append({"label": "購買", "url": "https://buy.example.com"})
# 刪除（例如刪掉「關於」）：
nav = [b for b in nav if b["label"] != "關於"]
# 位移（例如把「購買」移到第一個）：
buy = next(b for b in nav if b["label"] == "購買")
nav.remove(buy)
nav.insert(0, buy)
# 改連結：
for b in nav:
    if b["label"] == "聯絡":
        b["url"] = "mailto:new@example.com"

# Step 3：patch 整個 nav 回 backend
mcp_tool_call("landing_ai_mcp", "patch_landing_config", {
  "user_token": token, "campaign_id": campaign_id,
  "config_json": json.dumps({"header_nav": nav})
})
```

**使用者沒想法時、主動建議**：
```python
suggested = mcp_tool_call("landing_ai_mcp", "get_suggested_buttons", {
  "user_token": token, "campaign_id": campaign_id
})
# 回傳 [{"label": ..., "url": ...}, ...] 依 LP 內容推薦的 header 按鈕組合
```

展示：
```
我幫你看了一下這個 LP、建議 Header 放這幾個按鈕、你挑要哪幾個（也可以改文字或連結）：

1. 🧾 產品特色 → #features（滑到特色區）
2. 💬 Q&A → #faq
3. 📞 聯絡我們 → mailto:contact@...
4. 🛒 立即購買 → [你的購買頁]

你要哪幾個？或講「這幾個都加」/「只要 1、4」/「再加一個『預約試用』」。
```

**❌ 禁用手法**：
- 直接 `patch_landing_config` 傳只含新按鈕的 `header_nav: [new_button]` — 這會**覆蓋**整個 nav 陣列、既有按鈕全消失
- 憑記憶組 `header_nav`、不先 `get_landing_page` 讀當前值

### 歷史版本瀏覽（不是 undo/redo、是 version list）

**Plugin 的 mental model 是版本列表、不是 undo/redo command stack**。每個 stripe 每次 `regenerate_stripe` 都會**保留舊版當歷史**、整條時間線都在。「撤銷」= **跳到歷史裡某個版本**、不是 pop 一層動作。

**對使用者講的時候**：
- ❌ 不要用「undo / redo / 撤銷 / 重做」這種語言（誤導使用者以為是 command stack）
- ✅ 用「退回之前的版本」/「挑之前生過的某一版」/「切換版本」

**唯一正確流程**：先列歷史、使用者挑、切換：

```python
# Step 1：列出該 stripe 的所有歷史版本
history = mcp_tool_call("landing_ai_mcp", "get_stripe_history", {
  "user_token": token, "campaign_id": campaign_id, "stripe_idx": 0
})
# 回傳 [{version_id, created_at, image_url, prompt_used, is_current, ...}, ...]

# Step 2：展示給使用者挑（縮圖 + 時間 + 標記目前版）

# Step 3：使用者挑完、切到那個版本
# （tool 實作依 backend、可能是 undo_stripe 多次 call、或 restore-to-version、
#  不重要、對 LLM 自己是 internal；對使用者永遠是「我幫你切到 v2」）
```

**呈現範本**：
```
這頁你 regenerate 過 {N} 次、以下是歷史版本——挑你想要的：

v1（最初版、{created_at}）         v2（{created_at}）              v3 ⭐ 目前這版
![]({v1.image_url})                ![]({v2.image_url})              ![]({v3.image_url})

回「用 v1」我就幫你切回去。或回「生新的一版」我再跑一次 regenerate（100 pts）。
```

**不要做**：
- ❌ 對使用者說「我 undo 一次」然後 call `undo_stripe`——使用者不知道退到哪版、也不知道有沒有 undo 過頭
- ❌ 只 call `undo_stripe` / `redo_stripe` 不列歷史——使用者看不到時間線、盲切
- ❌ 假設使用者記得他改過幾次——可能只記得「某個版本比較好」、不記得是哪一次

### SEO / Metadata / FAQ

| User says | Tool |
|-----------|------|
| "Update the page title" / "改 SEO" / any SEO intent | `run_seo_optimize` (AI 一鍵全自動，見上方 Page-Level Edits 表) |
| "Edit the FAQ section" | `update_faq_content` |

## Editing Workflow

### Step 1: Understand the request + LOOK AT THE IMAGE
Parse what the user wants. **You MUST look at the current stripe image** (via `download_stripe` or the sales page) to understand the visual layout before deciding which tool/params to use.

**Decision tree for regeneration:**
- User wants to **change text content only** (no visual change) → use `update_stripe_texts`
- User wants to **remove or modify a specific visual element** → use `regenerate_stripe` with `rect_annotations_json` to red-box the exact area
- User wants a **full visual redesign** → use `regenerate_stripe` with `user_feedback`

### Step 2: Use red-box annotations for precision (IMPORTANT)

When the user points to a specific area ("remove that subtitle", "move this text", "delete that element"), you MUST:

1. **Look at the stripe image** to identify the element's position
2. **Calculate normalized coordinates** (0.0-1.0) for the target area
3. **Use `rect_annotations_json`** to mark it precisely

Example — removing a repeated subtitle at the bottom 30% of the stripe:
```
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7,
  "user_feedback": "移除紅框內的重複副標題文字",
  "rect_annotations_json": "[{\"x\": 0.05, \"y\": 0.65, \"width\": 0.9, \"height\": 0.15, \"color\": \"#EF4444\", \"notes\": \"移除此區域的重複文字\"}]",
  "mandatory_text_overrides_json": "{\"subheadline\": \"\"}"
})
```

**DO NOT** rely solely on natural language like "remove the subtitle" — the AI image generator needs visual coordinates to know WHERE to act.

### Step 3: Execute the edit
Call the appropriate MCP tool with exact parameters.

### Step 4: Verify the result
```
mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": edited_stripe_idx
})
```

### Step 5: Present and iterate
```
Updated stripe [N]: [description of change]

Want to:
A) Make more edits
B) See a preview of the full page → https://landingai.info/{locale}/lp/{campaign_id}
C) Open the visual editor → https://salecraft.ai/{locale}/editor?id={campaign_id}
D) Done — proceed to homepage building (/salecraft-homepage)
```

## Crop & Visual Adjustments

### Crop a stripe — 明確流程

**Schema**（驗證自 `Service_system/landing_ai_mcp/tools/landing_pages.py`）：

```python
crop_stripe(
  user_token, campaign_id, stripe_idx,
  top_px: int = 0,        # 從頂部砍幾 px（CropEditor 座標系、見下）
  bottom_px: int = 0,     # 從底部砍幾 px（CropEditor 座標系）
  image_width: int = 0,   # optional、傳實際圖寬給 backend 做精確 scaling
  image_height: int = 0   # optional、傳實際圖高
)
```

**4 個關鍵真相**：

1. **獨立參數**、**不是** `crop_json` JSON wrapper
2. **只能上下裁切**（`top_px` / `bottom_px`）、**沒有** left/right、沒有左右裁切能力
3. **px 值在 CropEditor 400×600 座標系**（不是實際圖片像素）：backend 把這個座標空間 scale 到實際圖、傳 `image_width` / `image_height` 給更精準的 scaling
4. **行為是累加**：每次 call 從 current 狀態（不是原圖）再裁、所以連續 call 會越切越小、`reset_crop` 才能回到原始

#### 主流程 — Vision Feedback Loop（LLM 自己 fetch 圖、看結果、iterate）

LLM 自己跑 **看圖 → 試裁 → 再看 → 不對就 reset + 重試** 的 loop、不要把翻譯位置的工作丟給使用者。

**前提**：LLM 環境支援 (a) authenticated HTTP fetch（Bash / Python sandbox / fetch with custom headers）、(b) vision（看 image 內容）。Claude Code / Claude.ai / ChatGPT-Plus / Gemini 多數都行。沒有這兩個能力時走下方 Fallback。

**Step 1 — Fetch 原圖、看圖、定位使用者要的元素**

```python
ref = mcp_tool_call("landing_ai_mcp", "download_stripe", {
  "user_token": token, "campaign_id": campaign_id,
  "stripe_idx": idx, "version": 0
})
# ref = {download_url, auth_header, content_type}

# 用 Bash / Python sandbox / fetch with header 拉圖：
#   curl -H "Authorization: <auth_header>" "<download_url>" -o /tmp/stripe.png
#   或 httpx.get(url, headers={"Authorization": auth_header})

# 用 vision 看 /tmp/stripe.png：
#   使用者說「保留到大拇指那條線」→ LLM 看圖、定位拇指輪廓在距頂端約 720 / 800 px 處
#   → 要砍掉的真實 px = 800 - 720 = 80
```

**Step 2 — 拿 stripe 真實尺寸 + 換算 CropEditor 600 座標**

```python
detail = mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token, "campaign_id": campaign_id, "stripe_idx": idx
})
real_h = detail["original_height"]   # 800
real_w = detail["original_width"]

# 真實 80 px → CropEditor 600 空間：80 / 800 × 600 = 60
crop_bottom_px = int(real_bottom_to_cut / real_h * 600)
```

**Step 3 — 呼叫 crop_stripe**

```python
mcp_tool_call("landing_ai_mcp", "crop_stripe", {
  "user_token": token, "campaign_id": campaign_id, "stripe_idx": idx,
  "top_px": 0,
  "bottom_px": crop_bottom_px,
  "image_width": real_w,
  "image_height": real_h
})
```

**Step 4 — 再 fetch 一次裁完的圖、用 vision 驗結果**

```python
# 重新 download_stripe → fetch → vision check
# 自問：使用者要的視覺元素（大拇指）有完整保留嗎？切太多嗎？切太少嗎？
```

**Step 5 — 不對就 iterate（reset + 重算 + 再裁）**

不要直接再 call `crop_stripe`（累加破壞、會越切越爛）。先 reset、再算、再裁：

```python
mcp_tool_call("landing_ai_mcp", "reset_crop", {...})   # 還原到原圖
# get_stripe_detail 確認回到原始 height
# 重新看原圖、調整 bottom_px、再 crop_stripe 一次
# 再 fetch + vision check
```

整個 loop 通常 **2-3 輪**收斂。LLM 自己跑、不打擾使用者。

**對使用者溝通**（Silent Execution）：
- 跑 loop 時別 narrate（不要講「我先 fetch 看看 / 我覺得砍 240 / 結果不對 / 我 reset」）
- 完成後一句話：「這頁裁好了、保留到你說的大拇指那條線、看看 OK 嗎？」
- 失敗到第 3 輪還收斂不了 → 停、告訴使用者「我裁出來都不太準、你直接看 LP、講大概保留上方多少 %」

#### Fallback — LLM 沒有 vision / 沒有 auth fetch 能力時

只有這時候才把翻譯工作交給使用者、要求他講百分比。Step 3-5 同上、只有 Step 1-2 改成：

**Step 1（Fallback）**：問使用者「保留上方多少 %？」
**Step 2（Fallback）**：用下表查 `bottom_px`、CropEditor 600 空間：

| 使用者要保留 | top_px | bottom_px |
|---|---|---|
| 上方 60% | `0` | `240` |
| 上方 80% | `0` | `120` |
| 上方 50%（上半）| `0` | `300` |
| 下方 40% | `360` | `0` |
| 中間（上下各砍 20%）| `120` | `120` |
| 不裁 | `0` | `0` |

### Reset crop to original
```
mcp_tool_call("landing_ai_mcp", "reset_crop", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 3
})
```

### Add soft fade edges between stripes

Creates smooth gradient transitions between consecutive stripes, giving the LP a more polished, seamless look instead of hard cuts between sections.

**Schema**（驗證自 backend 源碼）：

```python
set_stripe_soft_edge(
  user_token, campaign_id, stripe_idx,
  enabled: bool,        # true = 套用、false = 移除
  percent: float = 0.0  # 0.0 = 硬切、1.0 = 最強柔邊融合（注意是 0.0-1.0 float、不是 0-100 整數）
)
```

獨立 flat 參數、**不是** `soft_edge_json` JSON wrapper。

**對應使用者語言**：
- 「柔邊最強」/「100%」/「最融合」→ `percent=1.0`
- 「微微柔一點」→ `percent=0.3` 左右
- 「關掉柔邊」/「0%」→ `enabled=false`（或 `percent=0.0`）

**Response 欄位**：可能回傳 `top_px` / `bottom_px` / `strength` 等 backend 內部換算後的值——**那是 status display、不是下次 call 的 input**。重新 call 時繼續傳 `percent`。

**When to suggest soft edges:**
- 兩頁背景色差很大（轉場斷裂感）
- LP 整體有「拼貼感」、需要視覺連貫
- 使用者說「讓它順一點」、「更滑順的轉場」

**使用者溝通**（見 JARGON #13）：
- ❌ 「我把 bottom_px 從 30 改到 150」「strength 變成 1.0」
- ✅ 「這頁的柔邊調到最強、跟下一頁會自然融合」

### Add dark/light overlay for text readability

Adds a semi-transparent color layer over the stripe background, making text more readable when placed over busy or bright backgrounds.

**Schema**（驗證自 backend 源碼）：

```python
set_stripe_overlay(
  user_token, campaign_id, stripe_idx,
  enabled: bool,            # true = 套用、false = 移除
  color: str = "",          # CSS 顏色字串（"#000000" 深色、"#FFFFFF" 淺色）
  opacity: float = 0.5      # 0.0 完全透明、1.0 完全不透明、0.3-0.5 典型
)
```

獨立 flat 參數、**不是** `overlay_json` JSON wrapper。

**對應使用者語言**：
- 「白字壓不住、加暗一點」→ `color="#000000", opacity=0.4`
- 「深色背景、要加亮」→ `color="#FFFFFF", opacity=0.3`
- 「拿掉 overlay」→ `enabled=false`

**When to suggest overlays:**
- White text on light/busy backgrounds → dark overlay
- Dark text on dark backgrounds → light overlay
- User says "I can't read the text" or "text is hard to see"

**使用者溝通**：
- ❌ 「opacity: 0.4 / color: #000000 / enabled: true」
- ✅ 「這頁加一層微微的深色濾鏡、字會變比較清楚、看起來像加了墨鏡一樣那種」

## SEO & Search Optimization — one-click AI

SEO 優化是**單一按鈕 + AI 全自動**的付費功能（500 pts，beta 期免費）。使用者沒有「手動填 title / description / keywords」的路徑——AI 會根據 LP 內容和 Strategist 產出，一次生齊 meta / Open Graph / JSON-LD / FAQ schema / llms.txt / GEO summary。

### 呼叫流程（2 步）

1. **（可選）先 preflight 查扣點與餘額**：
   ```
   mcp_tool_call("landing_ai_mcp", "seo_preflight", {
     "user_token": token,
     "campaign_id": campaign_id
   })
   → { credit_cost, will_charge, current_balance, has_cached, sufficient }
   ```

2. **執行優化**：
   ```
   mcp_tool_call("landing_ai_mcp", "run_seo_optimize", {
     "user_token": token,
     "campaign_id": campaign_id,
     "force": False   # True 時強制重新生成（即使已有快取結果）
   })
   → 返回完整 SEO 套件，自動寫入 landing_page_config["seo_optimization"]
   ```

### 一次產出包含（AI 自動生成，不用使用者填）

| 輸出欄位 | 用途 |
|---|---|
| `meta_title` / `meta_description` / `meta_keywords` | Google 搜尋結果標題 + 摘要 |
| `og_title` / `og_description` / `og_type` | Facebook / LINE / Twitter 分享預覽 |
| `schema_article` / `schema_organization` / `schema_product` / `schema_faq` | JSON-LD 結構化資料（Google rich results） |
| `faq_items` | Q&A 陣列，供 FAQ schema + "People Also Ask" 使用 |
| `llms_txt_content` | `/llms.txt` 讓 AI 爬蟲（ChatGPT / Perplexity / Claude）更好引用 |
| `geo_summary` | 給 AI 搜尋引擎的**實體關聯摘要**（GEO = Generative Engine Optimization） |
| `canonical_url_path` / `robots_directive` | 規範性與索引控制 |
| `entity_connections` | 品牌實體關聯（wikidata-style） |
| `seo_score` / `recommendations` | AI 自評分數 + 改進建議 |

### 什麼時候主動推薦 SEO

- 使用者對 LP 視覺滿意後，**主動問一次**：「要不要順手把 SEO 做了？AI 一鍵全自動、beta 期免費」
- 使用者講「想讓 Google / ChatGPT 搜得到」、「上架前準備」、「怕沒人看到」
- 使用者準備嵌到自家官網前（SEO 要先做好再嵌）

### FAQ 單獨編輯（不跑完整 SEO 時）

若使用者只想改 FAQ 內容（例如覆蓋 AI 生成的 FAQ 用自己的答案）：

```
mcp_tool_call("landing_ai_mcp", "update_faq_content", {
  "user_token": token,
  "campaign_id": campaign_id,
  "faq_json": "[{\"question\": \"...\", \"answer\": \"...\"}]"
})
```

但**多數情況**直接跑 `run_seo_optimize` 更省事——AI 會連 FAQ 一起生。

**When to proactively suggest SEO:**
- After LP generation is complete and user is satisfied with the visuals
- When user mentions "search", "Google", "SEO", or "marketing"
- Before publishing — SEO should be set up before the page goes live

## Stripe Regeneration (3-Step Process)

Regeneration is async: **trigger → poll → confirm**.

### Step 1: Trigger — `regenerate_stripe`

5 regeneration modes (can combine):

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `user_feedback` | Natural language instruction | `"把標題改成紅色"` |
| `rect_annotations_json` | Red-box annotations on specific areas | `[{"x":0.1,"y":0.05,"width":0.8,"height":0.15,"notes":"放大這段文字"}]` |
| `reference_image_urls_json` | Reference images for style | `["https://example.com/style.jpg"]` |
| `mandatory_text_overrides_json` | Force-override text content | `{"headline":"新標題","subheadline":""}` |
| `typography_overrides_json` | Font/color overrides | `{"headline_color":"#00C853","headline_size_multiplier":1.5}` |

```
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7,
  "user_feedback": "移除副標題，只保留主標題",
  "mandatory_text_overrides_json": "{\"headline\": \"新標題\", \"subheadline\": \"\"}"
})
→ Returns: { "message": "Stripe 7 queued", "is_regenerating": true, "iteration": N }
```

**Note**: Coordinates in `rect_annotations_json` use normalized 0.0-1.0 range.

### Step 2: Poll — `get_stripe_regen_status`

Wait 10-30 seconds, then poll:
```
mcp_tool_call("landing_ai_mcp", "get_stripe_regen_status", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7
})
→ Check for: processing / completed / failed
```

### Step 3: Confirm — `complete_regeneration` (MANDATORY)

**You MUST call this to apply the result.** Without it, the new image won't show.
```
mcp_tool_call("landing_ai_mcp", "complete_regeneration", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7
})
→ Returns: { "message": "Stripe 7 regeneration completed" }
```

To cancel instead: use `cancel_stripe_regen` with the same params.

**Cost**: Free regenerations within quota (typically 8), then credits per regen.

### Uploading Reference Images for Regeneration

When the user provides a local image as style reference for regeneration:

```
# Step 1: Get signed upload URL
mcp_tool_call("landing_ai_mcp", "get_asset_upload_url", {
  "user_token": token,
  "brand_id": brand_id,
  "filename": "reference-style.jpg",
  "asset_type": "product",
  "content_type": "image/jpeg"
})
→ { "upload_url": "https://...", "public_url": "https://..." }

# Step 2: Upload via curl
# bash: curl -X PUT -H "Content-Type: image/jpeg" -T "/path/to/file.jpg" "{upload_url}"

# Step 3: Use public_url in regeneration
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 3,
  "user_feedback": "Match the style of the reference image",
  "reference_image_urls_json": "[\"https://...public_url...\"]"
})
```

This also works for user_suggestion_images in regeneration.

### Troubleshooting Regeneration

If `get_stripe_regen_status` always shows `is_regenerating: false` and `regeneration_started_at: null`:
- The regeneration worker may not be running on the backend
- Check backend Cloud Run logs for worker errors
- Try regenerating from the frontend UI to compare behavior
- This is a backend infrastructure issue, not an MCP issue

## Export Options

When user is done editing:

### Export as HTML
```
mcp_tool_call("landing_ai_mcp", "export_html", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: self-contained HTML string
```

### Export stripe image
```
mcp_tool_call("landing_ai_mcp", "download_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
→ Returns: { "download_url": "https://...", "auth_header": "Bearer ...", "content_type": "image/png" }
// Fetch the download_url with the auth_header to get the actual image file.
```

## Batch Edits Principle (CRITICAL — save user credits and time)

**Always batch as many changes as possible into ONE regeneration call per stripe.**

Regeneration is expensive (credits + wait time). Before calling `regenerate_stripe`:

1. **Collect ALL requested changes for this stripe** — text edits, visual removal, style changes
2. **Combine into ONE call** with:
   - `user_feedback`: natural language describing ALL changes
   - `rect_annotations_json`: ALL red/green boxes for every area to modify/preserve
   - `mandatory_text_overrides_json`: ALL text changes at once
   - `typography_overrides_json`: ALL style changes at once
3. **Never regenerate twice** for changes that could be batched

### Red Box Colors

| Color | Meaning | Use When |
|-------|---------|----------|
| `#EF4444` (red) | Remove / modify this area | Deleting elements, fixing errors |
| `#22C55E` (green) | Keep this area unchanged | Protecting elements from being affected |
| `#3B82F6` (blue) | Move / reposition this area | Relocating elements |

### Multi-Box Example (real scenario)

User says: "第 8 頁有重複標題，還有 debug 文字要移除"

ONE call handles everything:
```
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7,
  "user_feedback": "1. 移除下方重複的標題 2. 移除字體 debug 資訊 3. 只保留上方大字標題和 body 內文",
  "rect_annotations_json": "[
    {\"x\": 0.02, \"y\": 0.62, \"width\": 0.96, \"height\": 0.10, \"color\": \"#EF4444\", \"notes\": \"移除重複標題\"},
    {\"x\": 0.02, \"y\": 0.72, \"width\": 0.96, \"height\": 0.05, \"color\": \"#EF4444\", \"notes\": \"移除 debug 文字\"},
    {\"x\": 0.02, \"y\": 0.05, \"width\": 0.96, \"height\": 0.30, \"color\": \"#22C55E\", \"notes\": \"保留此主標題\"}
  ]",
  "mandatory_text_overrides_json": "{\"subheadline\": \"\"}"
})
```

WRONG — two separate calls for the same stripe:
```
// Call 1: remove subtitle
regenerate_stripe(stripe_idx=7, user_feedback="移除副標題")
// Call 2: remove debug text  ← WASTES credits + time
regenerate_stripe(stripe_idx=7, user_feedback="移除 debug 文字")
```

## Conversation Tips

- Always confirm which stripe before editing: "I'll edit stripe 2 (the features section). Correct?"
- After each edit, briefly describe what changed
- After any regeneration, remind user history is saved: 「這版不好的話隨時切回去——我可以列這頁生過的所有版本給你挑」
- **Collect ALL edits for a stripe before regenerating** — ask "Any other changes for this stripe?"
- For major changes, suggest regeneration over manual editing
- **Remind users about screenshot editing** — "You can also take a screenshot, circle what to change, and paste it here"
- **Provide the sales page and editor links** after every significant edit so users can verify visually

## SaleCraft Scope & Pricing (MUST READ)

### Who We Serve
SaleCraft is for **physical product sellers only** (skincare, food, fashion, health, electronics, etc.).
- ✅ Physical products, single items, single purpose
- ❌ Software/SaaS, multi-purpose platforms, abstract services

If the user's product doesn't fit, politely redirect:
> "SaleCraft 主要服務實體產品的行銷。你的需求可能更適合其他方案。"

### Pricing — Tell Before You Act
**1 USD = 30 pts | Minimum top-up: $20 = 600 pts**

Stripe regeneration has a **free allowance** before credits start being deducted:
- Every LP ships with `free_limit` free regenerations (equal to the page count — e.g. a 10-page LP has 10 free regens)
- Only after `free_remaining` hits 0 does each regeneration cost 100 pts
- Text-only edits (`update_stripe_texts`, `update_stripe_text_styling`) are always free

**DO NOT estimate cost from memory.** Always check the real `regen_quota` first.

### Regen cost — 2-step flow (MANDATORY before quoting any regen price)

**Step 1: Read current quota from `get_landing_page`**

Before quoting any regen price, fetch the LP config and read `regen_quota`:

```
mcp_tool_call("landing_ai_mcp", "get_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ config.regen_quota = {
    "free_limit": 10,           # total free regens for this LP
    "used_count": 2,            # regens consumed so far
    "free_remaining": 8,        # remaining free regens
    "credit_cost_per_regen": 100,  # cost once free is exhausted
    "is_paid_phase": false      # true once free_remaining == 0
}
```

**Step 2: Quote the user using `regen_quota`, not guesswork**

- If `free_remaining > 0`: "This regen is **free** (you have {free_remaining} free regens left)"
- If `is_paid_phase == true`: "This regen costs **{credit_cost_per_regen} pts**. Proceed?"

#### Consecutive regens — roll quota from each API response

If the user asks to regenerate multiple stripes in one turn (e.g. "redo stripe 3 and stripe 5"), **do NOT** use the Step 1 initial quota for both estimates. Each `regenerate_stripe` response returns fresh `free_remaining` / `credits_deducted` — use those to update your local snapshot before quoting the next stripe:

```
# MCP response can be str (JSON text) or dict (structured content).
# Parse defensively; do NOT write json.loads(r) unconditionally.
def _parse(resp):
    if isinstance(resp, dict):
        return resp
    if isinstance(resp, str):
        try:
            return json.loads(resp)
        except json.JSONDecodeError:
            return {}
    return {}

# initial snapshot: free_remaining = 2 (from get_landing_page)
# user asked to regen stripe 3, stripe 5

# stripe 3
r1 = mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {...stripe_idx: 3})
p1 = _parse(r1)
deducted_1 = p1.get("credits_deducted", 0)
free_remaining = p1.get("free_remaining", max(0, free_remaining - 1))

# stripe 5 — use updated free_remaining from r1, not the stale initial value
r2 = mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {...stripe_idx: 5})
p2 = _parse(r2)
deducted_2 = p2.get("credits_deducted", 0)
free_remaining = p2.get("free_remaining", max(0, free_remaining - 1))

# Report actual deducted pts per stripe (source of truth = backend response)
告知：「stripe 3 實扣 {deducted_1} pts，stripe 5 實扣 {deducted_2} pts；
目前免費剩 {free_remaining} 次。」
```

**Key rule**: per-stripe actual cost = that stripe's API response `credits_deducted`. Never predict or announce a price without the response in hand — under race conditions the backend is the source of truth.

**Top-up URL**: https://salecraft.ai/{locale}/marketingx

Before ANY paid action:
1. Read `regen_quota` via `get_landing_page` (don't guess from memory)
2. Check their balance: `get_me(user_token)` → `credits`
3. Tell the user the real cost based on current quota
4. If insufficient, guide them to top-up URL
5. Get explicit confirmation before proceeding

### Free Consultation Available
If the user seems unsure or is exploring, suggest the free consultation first:
> "If you'd like, I can do a free marketing consultation first — just say 'I want a consultation' or use the `saleskit` skill."

---

## Transition Prompts (MANDATORY — show at every decision point)

### Before editing starts:
```
準備編輯你的 LP！你可以用以下方式告訴我要改什麼：

1. 📸 截圖標註 — 截圖後用手機/電腦畫圈，貼過來我自動修
2. 💬 直接說 — 如「把標題改成 XXX」「第三頁背景太暗」
3. 🖥️ 用圖像編輯器 — 開啟 [editor_url] 自己拖拉修改
4. 🔍 SEO 優化 — 讓我優化搜尋引擎設定
```

### After each edit:
```
✅ 已修改 Stripe [N]：[描述]

繼續操作：
1. ✏️ 繼續編輯這一頁
2. 📄 編輯其他頁面
3. ↩️ 復原剛才的修改
4. 📸 截圖標註新的修改
5. ✂️ 裁剪 / 調整柔邊 / 加遮罩
6. 🔄 AI 重新生成這一頁（大幅修改時用）
7. 👀 預覽完整 LP → [sales_page_url]
8. ✅ 編輯完成，進入下一步
```

### When editing is done:
```
LP 編輯完成！接下來：

1. 🏠 建立品牌首頁 — 把 LP 嵌入完整網站
2. 📤 社群發佈 — 發到 IG / FB / TikTok
3. 📊 投放廣告 — Meta / Google Ads
4. 🌐 其他語言版本 — **開新 session 重新生一份**（plugin 沒有翻譯既有 LP 的工具）
5. 📥 匯出 — 下載 HTML / 圖片
6. 🔗 分享連結 — 產生分享用的短網址
```
