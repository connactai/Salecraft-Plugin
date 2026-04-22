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

### 🔴 MANDATORY Entry Check — Quality Gate 必須已跑完、否則回頭

**進 Phase 1 `generate_ta_options` 之前、先 `get_session` 驗證 Step 3 Quality Gate 跑過了**：

```python
session = get_session(session_id)
shared = session["wizard_shared_data"] or {}
files  = session["wizard_shared_files"] or {}

# 有產品圖的情況：Quality Gate 必須已跑
has_product_images = bool(files.get("product_images") or shared.get("product_images"))
if has_product_images:
    if not session.get("image_censor_results"):
        # Quality Gate 沒跑 → 不准 call generate_ta_options
        # 回頭跑 brand-onboard Phase 3.9：
        #   validate_images + analyze_image + digitize_product_text
        raise SkipGuard("Quality Gate 沒跑、回 brand-onboard Phase 3.9")

    latest = session["image_censor_results"][-1]
    if not latest.get("overall_passed") and not shared.get("_quality_gate_override"):
        # 品質不過、又沒使用者明確 override → 不准往下
        raise SkipGuard("Quality Gate overall_passed=false、且使用者沒 override、回 brand-onboard 給使用者看 issue")

# 無產品圖：必須有 _quality_gate_skipped_no_images flag
else:
    if not shared.get("_quality_gate_skipped_no_images"):
        # 沒明確標記 = 不確定是否該跳過、回頭補寫 flag 或補傳圖
        raise SkipGuard("無 image_censor_results 也無 skip flag、回 brand-onboard 確認")
```

**違反後果**：TA 用「沒被理解過的圖」的 brand 資料產生、Architect 瞎猜圖片內容、LP 出來招牌菜放錯段落 / 內裝照放產品介紹 → 使用者退費。

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

### 🔴 MANDATORY FIELD PRESENTATION — 不准過濾 `spokesperson_prompts`

`generate_ta_options` 每組 TA 回傳的 response **包含 `spokesperson_prompts` 陣列（通常 2 個候選）**、這**必須**逐組展示給使用者、每組 TA 獨立挑。LLM 常犯：整理成表格時只擷取 ta_name / 年齡 / 場合 / 色系 / 適合度、把 spokesperson 當次要資訊砍掉 → 使用者不知道有代言人可以挑 → 生完 LP 才發現代言人是 LLM 配的。

**代言人是 LP hero 首屏最關鍵的視覺** — 直接決定受眾「我是不是他」的第一秒反應。不可省。

每組 TA 的展示格式**必須包含 spokesperson 區塊**（見下方範本 `👤 代言人候選` 子項）。

### 🔴 Anti-fabrication rule

**DO NOT write TA names / descriptions inline from your own imagination.**

- ❌ Listing「商務宴客、精緻餐飲愛好者、竹科外商」in a flat sentence as if 3 TAs — that's fabrication, not TA selection. Looks like a list but user can't pick, compare, or see demographics.
- ✅ The **only** acceptable source of TA candidates at this Phase is the `generate_ta_options` tool's return value.
- If you haven't called it yet — call it now before proceeding.
- Fabrication produces categorical-level strategy (same failure mode as skipping the Product Concreteness Gate in `saleskit`).

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
   👤 代言人候選（挑 1 個或說「都不用」）：
      A. [spokesperson_prompts[0] 描述 — 例如「35-45 歲儒雅男性、西裝、都會背景」]
      B. [spokesperson_prompts[1] 描述 — 例如「28-35 歲都會女性、casual chic、咖啡廳」]

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
   👤 代言人候選（挑 1 個或說「都不用」）：
      A. [spokesperson_prompts[0] 描述 — 例如「35-45 歲儒雅男性、西裝、都會背景」]
      B. [spokesperson_prompts[1] 描述 — 例如「28-35 歲都會女性、casual chic、咖啡廳」]

   🧠 Strategic value: This audience is driven by aspiration and lifestyle identity.
   Best if: Your product represents a lifestyle upgrade, premium positioning,
   or status signal. Luxury, wellness, and personal growth products thrive here.
   Tone: Aspirational, visual-heavy, storytelling.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

3. 🌿 [TA Name] — Appeal: MEDIUM
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]
   👤 代言人候選（挑 1 個或說「都不用」）：
      A. [spokesperson_prompts[0] 描述 — 例如「35-45 歲儒雅男性、西裝、都會背景」]
      B. [spokesperson_prompts[1] 描述 — 例如「28-35 歲都會女性、casual chic、咖啡廳」]

   🧠 Strategic value: This is a secondary audience that can expand your reach.
   Best if: You are running multi-channel campaigns and want to capture
   adjacent segments. Lower conversion intent but larger volume.
   Tone: Educational, approachable, curiosity-driven.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

4. 🏢 [TA Name] — Appeal: MEDIUM
   Who: [1-line description, demographics]
   Angle: "[fabt_translation]"
   Style: [visual_style] / [language]
   👤 代言人候選（挑 1 個或說「都不用」）：
      A. [spokesperson_prompts[0] 描述 — 例如「35-45 歲儒雅男性、西裝、都會背景」]
      B. [spokesperson_prompts[1] 描述 — 例如「28-35 歲都會女性、casual chic、咖啡廳」]

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

## Phase 2.5: Per-TA Spokesperson Image Preview (MANDATORY if any TA has spokesperson)

### 🔴 Free quota, NOT 500 pts — 不准誤導使用者扣款

`generate_ta_spokesperson` **不扣使用者點數**——它走**帳號級免費配額**（一般每期有幾次）、用 `get_spokesperson_generation_status` 查剩餘。**絕對禁止**對使用者說「要生代言人要扣 500 pts / 個」這種話——那是**錯的**、會讓使用者放棄代言人或以為被多扣。pricing table 上的 "500 pts" 若出現、以 tool 實際行為為準（`get_spokesperson_generation_status` 回傳的 quota-based 結構）。

### 🔴 進 Phase 2.5 第一件事：查配額、告訴使用者剩幾次

```python
status = mcp_tool_call("landing_ai_mcp", "get_spokesperson_generation_status", {
  "user_token": token
})
# 回傳 {used: 2, limit: 5, remaining: 3}
```

**必須**把 `remaining` 用人話告訴使用者、再開始 preview 流程：

```
你選的兩組 TA 都要代言人——AI 代言人生成是免費的（帳號額度、不扣點數）、
你這期還剩 {remaining} 次。我接下來幫每組生一版預覽給你看：
  - 不滿意可以「重生」（再用一次配額）或「調整 XXX」（改 prompt 再跑）
  - 也可以「換成候選 B」或「改用上傳照片」或「都不用」

先從 TA 2「質感美學獵尋者」開始（你挑的候選 A 品味策展人）……
```

**額度不足時的 fallback**（`remaining < 使用者選的 TA 數量`）：
```
提醒一下、你這期帳號剩 {remaining} 次 AI 代言人生成配額、但你挑了 {N} 組 TA。
可以怎麼辦：
  1. 先生 {remaining} 組（挑你最在乎的 TA）、其他組改上傳自己照片或不用人物
  2. 全部都改上傳照片（我可以教你怎麼傳）
  3. 全部都不用、代言人空缺、LP 用料理 / 場景圖當主視覺（省時也不一定差）

你想走哪條？
```

### 🔴 文字描述 ≠ 可批准的代言人

使用者從 `spokesperson_prompts` 挑了候選 A 或 B 之後、LLM **絕對不准**直接把那段文字描述當成已確認、拿去 Cost 複誦扣點。原因：

> 「一位 50-60 歲午夜藍西裝、沉穩自信的成熟領導者」這段文字能產生 10 種截然不同的實際視覺——臉型、族裔、氣質、背景、打光、表情差異巨大。使用者沒看過實際照片就點頭 = 盲簽合約、生完 LP 才發現「這個人不是我要的」→ 退費。

**必須呼叫 `generate_ta_spokesperson` 拿 front_url + side_url 實際圖、markdown 展示給使用者、等逐組點頭、才能進 Phase 3。**

### 🔴 Step 0（必先跑）— **主動**查 brand 資產庫 + **主動推薦** reuse

**絕對禁止**等使用者問「有沒有舊的代言人可以用」才查——LLM 必須**自己主動**先查、**主動推薦**既有資產當預設方案、讓使用者要「生新的」時要有理由。

**為什麼主動推薦 reuse**（要對使用者講清楚）：
1. **brand 一致性**：同一個代言人跨 LP / Reels / 廣告、受眾記得住「這個人 = 這個品牌」、比每次換人強
2. **配額**：既有資產不吃 AI 生成配額、留著做其他更重要的新 TA
3. **時間**：reuse 立刻可用、新生要 30-60 秒 + 可能要重生調整

**進 Phase 2.5 的第一個動作**（不旁白、靜默跑）：

```python
existing = mcp_tool_call("landing_ai_mcp", "list_spokespersons", {
  "user_token": token, "brand_id": brand_id
})
# 回傳 [{spokesperson_id, name, description, photo_urls, is_ai_generated, created_at}, ...]
```

**若資產庫非空**：**帶推薦立場展示**、不是中立列選項。例：

```
你 {brand_name} 的資產庫裡有 {N} 個代言人可以用——**我建議先從這邊挑**（省配額、brand 一致性強、立刻能用）：

1. ⭐ {name_1}（{description 前 40 字}）{若 created_at 新加「最近剛做過」}
   ![]({photo_urls[0]})
2. {name_2}（{description}）
   ![]({photo_urls[0]})
…

我的推薦：
- TA 1「{ta_1_name}」→ 用 #{best_fit_id} {reason、例：「這位西裝商務氣質跟巔峰企業家 TA 最搭」}
- TA 2「{ta_2_name}」→ 用 #{best_fit_id} {reason}

你可以：
- 回「照你建議」→ 直接套用我推的配對、跳過生成
- 回「TA1 用 #X、TA2 用 #Y」→ 自己挑配對
- 回「TA1 用 #X、TA2 重新生」→ 部分 reuse + 部分新生
- 回「都要新生」→ 走生成流程（**但請告訴我為什麼**——既有不夠像嗎？想換風格？）
```

**要點**：
- 用 ⭐ / 「我的推薦」明示 LLM 的 stance、不是中立列表
- 若 LLM 能判斷資產庫哪位最適合哪個 TA（看 description + TA 受眾）、**主動配對**、不要叫使用者自己看
- 使用者選「都要新生」要**反問原因**（「既有不夠像嗎？」）、避免 FOMO 心理白燒配額

**若資產庫為空**：靜默略過這段、直接走下方生成流程、但要跟使用者講「這是你 {brand_name} 的第一批代言人、生完我會存進資產庫、之後做新 LP 可以 reuse」。

### 流程（per-TA 獨立跑、不合併）

對每個需要**新生**代言人的 TA group（非 reuse 既有資產的）：

```python
# 每個 TA 各自跑一輪（TA 1 完再 TA 2、不平行 preview 避免使用者 overload）
for ta in tas_needing_new_spokesperson:
    chosen_prompt = ta.spokesperson_prompts[0 if ta.spokesperson_choice == "A" else 1]

    # 先查剩餘配額（每輪都查、因為前一輪可能剛用掉一次）
    status = mcp_tool_call("landing_ai_mcp", "get_spokesperson_generation_status", {
      "user_token": token
    })
    if status["remaining"] <= 0:
        # 告訴使用者「這期 AI 代言人配額用完、選上傳照片 or 用資產庫 or 不用人物」
        break

    # generate（~30-60 秒、會等）
    result = mcp_tool_call("landing_ai_mcp", "generate_ta_spokesperson", {
      "user_token": token,
      "prompt": chosen_prompt,
      "ta_name": ta.ta_name
    })
    # result.images = { front_url, side_url }
    # result.spokesperson_id = "sp_xxx"（這是 generation 的 id、還沒登記進 brand 資產庫）
```

### 🔴 展示 + 等使用者點頭（per-TA 一輪）

```
TA 1「巔峰企業家」的代言人候選 A（卓越領導者）我生出來了、你看看：

**正面**
![front]({result.images.front_url})

**側面**
![side]({result.images.side_url})

這個可以嗎？
- 回「OK」→ 我把這位定案、接著生 TA 2 的
- 回「重生」→ 同 prompt 再跑一次（剩 {status.remaining - 1} 次配額）
- 回「調整 XXX」→ 你講要改什麼（眼鏡 / 膚色 / 氣質 / 場景 …），我修 prompt 重跑
- 回「換成 B」→ 改用候選 B「高階經理人」重新生
- 回「都不用」→ 這組 TA 不要代言人、改用場景 / 產品圖當主視覺
- 回「我要用自己的照片」→ 走**上傳路徑**（見下方、**跟 AI 生成是替代關係、不共存**）
```

### 上傳自己照片的分支流程（per-TA、非 AI 生成）

若使用者對某組 TA 選「改用上傳照片」、或是 AI 配額用完要 fallback：

```python
# Step 1: 接使用者傳的照片（pasted base64 或給 URL）
if base64_data:
    upload_result = mcp_tool_call("landing_ai_mcp", "upload_base64", {
      "user_token": token,
      "brand_id": brand_id,
      "filename": "spokesperson.png",
      "base64_data": base64_data,
      "asset_type": "spokesperson",
      "content_type": "image/png"
    })
    photo_url = upload_result["public_url"]
elif user_given_url:
    photo_url = user_given_url   # 公開可存取 URL、跳過上傳

# Step 2: 登記進 brand 資產庫（跟 AI 路徑收尾同步、拿到 spokesperson_id）
registered = mcp_tool_call("landing_ai_mcp", "create_spokesperson", {
  "user_token": token,
  "brand_id": brand_id,
  "name": ta.ta_name + " 代言人（自備）",
  "description": f"User-uploaded spokesperson photo for TA「{ta.ta_name}」",
  "photo_urls_json": json.dumps([photo_url]),
  "is_ai_generated": False
})

# Step 3: 寫進 per-TA session（同 AI 路徑）
update_session(data={
    "wizard_ta_groups": [{"id": ta.id, "spokesperson_id": registered["spokesperson_id"], ...}]
})
```

**為什麼也走 `create_spokesperson` + `spokesperson_id`、不是直接寫 `wizard_ta_group_files[].spokesperson_faces[]`**：
- 統一 reuse 路徑、使用者下次做 LP/Reels/廣告時一樣看得到這位上傳代言人
- `wizard_ta_group_files[].spokesperson_faces[]` 是 legacy 備援路徑、新流程都走 `spokesperson_id`

### 🔴 使用者 OK 後的收尾（兩步、不准省）

**Step A — `create_spokesperson` 登記進 brand 資產庫**

使用者點「OK」之後、**必須**呼叫 `create_spokesperson` 把這個 approved 的代言人登記進 brand 的資產庫——這樣下次開新 LP、做 Reels、跑廣告時都可以 reuse、不用再吃配額。**禁止**跳過這步直接 update_session、會造成「這個代言人只這次用、下次不見」。

```python
registered = mcp_tool_call("landing_ai_mcp", "create_spokesperson", {
  "user_token": token,
  "brand_id": brand_id,
  "name": ta.ta_name + " 代言人",   # 或讓使用者自訂名字
  "description": f"AI-generated for TA「{ta.ta_name}」. Prompt: {chosen_prompt[:100]}",
  "photo_urls_json": json.dumps([result["images"]["front_url"], result["images"]["side_url"]]),
  "is_ai_generated": True
})
# 回傳 {spokesperson_id: "sp_xxx", ...}
```

告訴使用者：
> 「已存進你 {brand_name} 的代言人資產庫、命名「{name}」、下次要做新 LP / Reels / 廣告可以直接挑、不會再吃配額。」

**Step B — 寫進 session（把 brand-level sp_id 綁到 per-TA）**

```python
update_session(data={"wizard_ta_groups": [
    {..., "spokesperson_id": registered["spokesperson_id"],
          "spokesperson_front_url": result["images"]["front_url"],
          "spokesperson_side_url": result["images"]["side_url"]}
    for ta in selected_ta_groups
]})
```

**Reuse 情境簡化**：若 Step 0 使用者直接挑了資產庫現有代言人、就跳過 Step A（已經在資產庫裡了）、只做 Step B、用既有的 `spokesperson_id` + `photo_urls`。

### 絕對禁止

- ❌ 拿 `spokesperson_prompts` 的文字描述進 Cost 複誦、當成使用者已經看過代言人 → 使用者其實沒看過實際圖
- ❌ 把 2+ 組 TA 的 preview 平行展示（「TA1 正面 / 側面、TA2 正面 / 側面」一次 4 張圖）→ 使用者 overload 會亂點、分辨不出對應哪組
- ❌ `generate_ta_spokesperson` 生完就直接 `create_spokesperson` 登記 → 使用者沒看過實際圖 = 盲簽
- ❌ 只展示 front_url 不展示 side_url → 側面是代言人在 LP hero 擺位的關鍵、兩張都要看

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

Available (8 canonical names — must match Wizard UI dropdown exactly):
  "Traditional Chinese (Taiwan)" / "English" / "Japanese" /
  "Spanish" / "Portuguese" / "Vietnamese" / "Indonesian" / "Thai"

⚠️ **寫入 `wizard_ta_groups[i].language` 必須用上面 canonical name、不是 ISO code**。
Wizard UI dropdown 的 SelectItem value 是 canonical name（見
`marketing_frontend_commercial/app/[locale]/wizard/page.tsx:5427-5434`）。
寫 `"en"` / `"zh-TW"` 等 ISO code 雖然 backend `_normalize_language` 能吃、但
Wizard UI dropdown 會顯示空白、使用者以為沒設定。實測確認 2026-04-22。

Notes:
- Traditional Chinese (Taiwan): Default for Taiwan market
- English: Best for international / tech audiences
- Japanese: Text 20-40% longer than zh-TW, use ですます体
- Spanish: ~30% longer than English; accent marks mandatory (á/é/í/ó/ú/ñ/ü)
- Portuguese: Default PT-BR unless brand specifies PT-PT; accents mandatory
- Vietnamese: Tone marks mandatory (à/á/ả/ã/ạ/ă/â/đ/ê/ô/ơ/ư); ~20% longer
- Indonesian: standard BI, avoid regional slang; use "Anda" formal
- Thai: no spaces between words, tone/vowel marks critical

**URL locale ≠ session language**: URL paths（`/zh-TW/login`、`ui_locale`）用 ISO code、是另一套系統。別混淆。

Not yet supported by Wizard UI dropdown (even though backend may accept):
  Simplified Chinese (zh-CN), Korean, French, Arabic, German, Malay, Hindi —
  寫進 `session.language` 後 UI dropdown 會顯示空白、使用者無法視覺確認。

Each language is a fully independent LP generation — no cheap "translation" path exists. If the user wants two languages, they pay for two full LP generations.
```

### 🔴 MANDATORY: reject `suggested_language: "Bilingual"` from `generate_ta_options`

Backend ta_generator prompt 仍包含 `Bilingual` 為合法 output（`marketing_backend/agents/ta_generator.py:283, 399`）、但下游 Wizard UI / `generate_session` / Strategist 都不認、寫入 = silent fallback 到 zh-TW 或中英混雜亂文案。

**規則**：

- 收到 `suggested_language: "Bilingual"` / `雙語` / `中英` → **絕對不寫入 session**
- 寫進 session 的 `language` 必須是 8 canonical name 白名單的單一值（見上方 4E Available 清單）、不是複合值、不是 ISO code、不是其他自創值
- 若 `generate_ta_options` 回傳 ISO code（例 `"en"`）→ 轉成 canonical name（`"English"`）再寫入 session、讓 Wizard UI dropdown 能對上
- 使用者若堅持「要雙語」→ 解釋等於 2 份獨立 LP 生成（2× 扣點）、每份鎖一個 locale（引用 4E「no cheap translation path exists」）
- 禁止靜默替使用者挑 locale（違反 FIRST-RESPONSE RULE「LLM 不替使用者選規格」）
- 禁止暗示「雙語 LP」這個產品存在——產品不存在

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

This skill is **FREE**. `generate_ta_options` / `generate_ta_spokesperson` / `get_spokesperson_generation_status` / `list_spokespersons` / `create_spokesperson` all run on free quota — they do NOT deduct user credits. The user only gets charged later when `generate_session` fires (at 200 pts per page per TA).

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
