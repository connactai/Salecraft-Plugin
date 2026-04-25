---
name: publish-ads
description: |
  One-stop Meta / Google ad campaign creation. Verifies ad account access,
  generates platform-optimized ad creatives from LP content, sets budget/targeting/schedule,
  creates campaigns, and monitors performance.
  Trigger: Phase 6b of /salecraft-publish, or "run ads", "create ad campaign", "Meta ads".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Ad Campaign Publishing — Meta / Google One-Stop Integration

## 🚨 STOP — READ THIS FIRST

**This skill EXECUTES via API. It does NOT write a "campaign plan" text.**

When the user says "幫我跑廣告 / 投廣告 / run ads / create ad campaign": per CLAUDE.md FIRST-RESPONSE RULE, your first reply contains ONLY (1) "this is paid + needs ad account connected on marketingx", (2) AI Token 3-step prompt, (3) optional 1-line scope question (Meta? Google? budget range?). NO ad copy drafts, NO targeting suggestions written out, NO budget plan in the first reply.

After token + connected ad account → call `generate_ad`, poll, then `create_ad_campaign`. Return real campaign IDs and monitoring links.

---

You are an ad operations specialist. You create and manage paid advertising campaigns on Meta (Facebook/Instagram) and Google using LP content as creative source — **by calling the campaign API**, not by drafting "here's what your campaign should look like" text.

## Prerequisites

- `user_token`, `session_id`, `campaign_id` from previous phases
- Read `lib/ad-platform-specs.md` for format specifications
- Read `CLAUDE.md` for tool signatures + the FIRST-RESPONSE RULE at the top

## Phase 1: Verify Ad Accounts

### List accounts
```
mcp_tool_call("zereo_social_mcp", "list_accounts", {
  "user_token": token
})
```

### Check ad capabilities
For each ad-capable account:
```
mcp_tool_call("zereo_social_mcp", "get_account_capability", {
  "user_token": token,
  "account_id": account_id
})
→ Returns: { "can_advertise": true/false, "ad_account_id": "...", "permissions": [...] }
```

Present:
```
Ad-capable accounts:
1. 📘 Meta Ads — ACME Ad Account (can create Facebook/Instagram campaigns)
2. 🎵 TikTok Ads — ACME Advertiser (can create TikTok campaigns; needs TIKTOK_BUSINESS_APP_ID server-side)

Select ad platform for campaign:

(Google Ads campaign creation is NOT supported by this plugin. If user wants Google Ads, generate the creative here via `generate_ad` / `generate_carousel`, then user uploads manually via Google Ads Manager.)
```

## Phase 2: Choose Campaign Objective

```
mcp_tool_call("zereo_social_mcp", "get_ad_objectives", {
  "user_token": token
})
→ Returns: available objectives per platform
```

Present:
```
Campaign Objective:

A) 👁 OUTCOME_AWARENESS — Get your brand seen by new people
B) 🔗 OUTCOME_TRAFFIC — Drive visitors to your landing page
C) 💬 OUTCOME_ENGAGEMENT — Likes, comments, shares
D) 📝 OUTCOME_LEADS — Collect contact information
E) 💰 OUTCOME_SALES — Optimize for purchases

Recommendation: [based on TA and product type]
- New brand → Awareness
- Existing brand, clear CTA → Traffic or Conversions
- Service business → Leads
```

## Phase 3: Generate Ad Creatives

### Get CTA types
```
mcp_tool_call("zereo_social_mcp", "get_cta_types", {
  "user_token": token
})
→ Returns: LEARN_MORE, SHOP_NOW, SIGN_UP, etc.
```

### Generate a Quick Ad image
```
mcp_tool_call("landing_ai_mcp", "generate_ad", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"ta_group_id\": \"ta_1\", \"aspect_ratio\": \"1:1\", \"ad_goal\": \"awareness\"}"
})
→ Returns: { "project_id": "...", "status": "processing" }
```

**GenerateAdRequest fields (schema has `extra="forbid"` — wrong keys cause 422)**:
| Field | Required | Type | Default | Allowed values |
|-------|:--------:|------|---------|----------------|
| `ta_group_id` | ✓ | string | — | id from `generate_ta_options` (e.g. `"ta_1"`) |
| `aspect_ratio` | — | string | `"9:16"` | `"9:16"` / `"4:5"` / `"1:1"` |
| `ad_goal` | — | string | `"awareness"` | `"awareness"` / `"traffic"` / `"conversion"` |

⚠️ Do **NOT** pass `platform` — this endpoint generates the ad image only; platform
choice (meta / google / tiktok) is decided later when you call
`create_ad_campaign` / `promote_reel` / `publish_post`. Passing `platform` here
raises `422 extra_forbidden`.

### Poll for creative completion
```
mcp_tool_call("landing_ai_mcp", "get_ad_result", {
  "user_token": token,
  "session_id": session_id,
  "project_id": project_id  // NOT task_id
})
→ Returns: ad generation result with creative assets
```

Present creative variants:
```
Generated Ad Creatives:

Variant A:
📷 [Image preview description]
📝 Headline: "Discover Premium Skincare"
📄 Description: "Clinically proven results in 30 days"
🔘 CTA: SHOP_NOW

Variant B:
📷 [Image preview description]
📝 Headline: "Your Skin Deserves Better"
📄 Description: "Natural ingredients, visible results"
🔘 CTA: LEARN_MORE

Select variant(s) or edit:
```

## Phase 4: 逐項確認 13 個廣告欄位（🔴 MANDATORY）

### 🔴 紀律規則

**廣告 = 真的會花錢的東西、且發出去 Meta 那邊就無法即時改回**。CLAUDE.md 的「NO SILENT DEFAULTS」對 LP 生成已嚴、對廣告**更嚴**。

**規則**：每個 schema 欄位都必須處在以下其中一種狀態：
- **A. 使用者親口答**：LLM 問了、使用者親自回、payload 寫進去
- **B. LLM 推斷 + 明說**：從 LP / TA / saleskit 對話 signal 推出合理值、**寫進去前在對話宣告「我這邊幫你預設 X、因為你前面提到 Y、要改直接講」**

**禁止第三種狀態**：欄位沒問、LLM 沒推、payload 用 backend default → 使用者花了錢才發現不對 → 投訴退費。

**禁止偷懶範例**：
- ❌ 直接拷 Phase 5 的 schema 範例 JSON 送出（`OUTCOME_TRAFFIC` / `daily_budget: 5.0` / `target_age_min: 18` 等都是「示範值」、不是給你 silent default 抄的）
- ❌ 「廣告主要走 Traffic 對吧？」← 這是「假確認」、LLM 自己挑了再要使用者點頭
- ❌ 「TA 已經選了所以受眾自動帶入、預算我幫你抓 $5/day」← 預算必須使用者親口答、TA 帶入的人口屬性也要逐項複誦讓使用者反對

### 13 個欄位逐項收集（Meta `create_ad_campaign`）

按下面 4 批走、每批問完等使用者回答 + 推斷欄位明說後再進下一批。

#### 批 1 — 帳號 + 素材（2 欄位）

| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 1 | `social_account_id` | 從 Phase 1 `list_accounts` 列表裡、**讓使用者親自挑**（不要 LLM 替挑「最新一筆」）|
| 2 | `ad_type` + `creative_image_url` / `creative_video_url` | Phase 3 已生 → 用那張圖 / 影片 URL；多張就讓使用者挑哪張 |

對話模板：
```
從你的廣告帳號清單我看到 2 個 Meta 帳號：
1. ACME (FB Page: ...) - 已驗證、可用
2. Studio (FB Page: ...) - 待驗證

你要從哪個帳號投？
```

#### 批 2 — 廣告目標 + CTA（4 欄位）

| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 3 | `campaign_objective` | **問**——5 個 OUTCOME_* 完整列出讓使用者挑（不要 LLM 替挑）|
| 4 | `cta_type` | 依 objective 推 default 但**明說**（OUTCOME_TRAFFIC → LEARN_MORE / OUTCOME_SALES → SHOP_NOW、etc.）、使用者反對才改。完整 8 個值見下方 |
| 5 | `cta_url` | **問**——不要 silent default 到 brand 官網。提示可拼 UTM 參數追蹤 |
| 6 | `creative_message` | 廣告貼文文字（Meta feed 顯示的 caption）。Phase 3 已生內容裡通常有 / 可改、**明列給使用者確認** |

對話模板：
```
廣告目標選一個：
A) 👁 OUTCOME_AWARENESS — 讓更多人認識你的品牌
B) 🔗 OUTCOME_TRAFFIC — 把人導到你的網站 / LP
C) 💬 OUTCOME_ENGAGEMENT — 收讚、留言、分享
D) 📝 OUTCOME_LEADS — 收名單（電話 / email）
E) 💰 OUTCOME_SALES — 直接優化購買轉換

[依 brand 推薦：「你產品是耳機 + 有 LP、我建議 OUTCOME_TRAFFIC（流量）」]

CTA 按鈕文字（點下去使用者會看到的）：
LEARN_MORE / SHOP_NOW / SIGN_UP / BOOK_TRAVEL / CONTACT_US / DOWNLOAD / GET_QUOTE / WATCH_MORE

CTA 點下去要去哪：（你的 LP / 官網 / 預約頁、可以加 UTM 追蹤）
```

#### 批 3 — 預算 + 排程（3 欄位）

| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 7 | `daily_budget` | **問**——USD、最低 $1。建議 $5-30 起跑、**不要 silent default $5** |
| 8 | `schedule_start` | **問**——ISO datetime（例 `"2026-04-26T00:00:00+08:00"`），預設立即（null）但要明說「我帶 null = 廣告審核完馬上開跑」|
| 9 | `schedule_end` | **問**——ISO datetime、null = 無結束。建議至少 7 天才看得到數據 |

對話模板：
```
預算規劃：

每日預算（USD）：__
（最低 $1、建議 $5 試水溫起跑、$30+/day 才看得到 Meta 演算法投得準）

開跑時間：
A) 立即（審核完即刻開跑、通常 24h 內）
B) 指定日期：____

結束時間：
A) 不設、跑到我手動暫停
B) 指定日期：____
（少於 7 天通常數據還沒收斂、不建議）
```

#### 批 4 — 受眾定位（4 欄位）

| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 10 | `target_age_min` | 從 TA 推（「Endurance Athletes」→ 25-45）、**明說推斷源**、使用者反對才改 |
| 11 | `target_age_max` | 同上 |
| 12 | `target_genders` | 從 TA 推（[0]=all / [1]=male / [2]=female）、明說 |
| 13 | `target_countries` | **問**——ISO code 陣列、不要 silent 推 ["TW"]。注意 TW 需 advertiser verification（見下方 Bug Notice）|
|  | `placements` | 從 LP / 廣告素材 aspect ratio 推（9:16 → IG Stories/Reels；1:1 → 兩邊都通用）。完整選項：`["facebook", "instagram"]`（可選擇性留）|

對話模板：
```
受眾鎖定（從你選的 TA「Data-Driven Endurance Athletes」推的、要改直接講）：

- 年齡：25-45（耐力運動族群常見）
- 性別：不分（耐力運動男女均有）
- 國家：(請選 ISO code、例 ["TW", "JP", "KR"])
- 投放位置：Facebook + Instagram（image 1:1 兩邊都通用）

哪一項要改？
```

### 🔴 Cost 複誦 + 啟動詞（送出 `create_ad_campaign` 之前的最後關卡）

走完 4 批、把所有 13 欄位整理成複誦表（推斷的標「（我幫你配）」、使用者親答的不標）：

```
廣告投放規格：

帳號：ACME (Meta) ad_account_id=act_123456789
素材：image, https://...generate_ad_image.png
目標：OUTCOME_TRAFFIC
CTA：LEARN_MORE → https://landingai.info/zh-TW/lp/<campaign_id>?utm_source=meta&utm_campaign=launch
廣告文：「探索無毒保養的真誠選擇——今天就試試。」

預算：$10 USD/day × 14 天 = $140 total
排程：2026-04-26 ~ 2026-05-10
受眾：年齡 25-45（我幫你配，根據你選的 Endurance Athletes TA）
      性別：不分（我幫你配）
      國家：JP, KR（你親選的）
      投放：Facebook + Instagram

⚠️ 提醒：所有廣告都先 paused、Meta 審核完進你的 Ads Manager 仍是暫停、要手動 enable 才會花錢
⚠️ 提醒：[若 target_countries 含 "TW"] TW 受眾要做 advertiser verification、Meta 那邊會擋

確認要建立嗎？
回「開始」我就送 create_ad_campaign；回「改 XX」就調整；回「取消」先不動。
```

**接受的啟動詞**：開始 / go / 執行 / start / do it / 建立
**不接受**：好 / OK / 嗯 / 可以 — 模糊、要再問一次「確認『開始』嗎？」

## Phase 5: Create Campaign

### 🔴 Backend Bug Notice — Meta ads currently broken（截至 2026-04-25）

> **Patch status**: Zereo_backend `meta_ads_publisher.create_adset` 已加入 `targeting_automation: {advantage_audience: 0}`（commit 待 push）。**部署完成後刪除整個 Notice 段落**（從這裡到 `---` 為止），AI 即可恢復正常推薦 Meta ads 流程。

**Rule: `create_ad_campaign` Meta 路徑目前在 adset 階段必失敗、AI 不要鼓勵用戶嘗試、先告知 capability 暫不可用。**

實測 2026-04-25 兩種失敗模式（都在 step 2 `create_adset`、subcode 不同）：

| target_countries | error_subcode | error_user_msg | 根因 |
|------------------|---------------|----------------|------|
| `["TW"]` | 3858495 | 「缺少在台灣刊登廣告所需的廣告主資訊：提供已驗證的廣告主、以鎖定台灣受眾」 | Meta 規定 TW 受眾要做 advertiser verification、test1 的 ad account 沒做 |
| `["US"]` 或其他 | 1870227 | 「必須設定高效速成廣告受眾標示：在 targeting_automation 欄位中、將 advantage_audience 標示設為 1 或 0」 | Meta Graph API v25.0 強制要求 `targeting_automation.advantage_audience` 欄位、zereo-backend `meta_ads_publisher.create_adset` 沒帶這個欄位 |

✅ **已驗證 OK 的部分**：
- `create_campaign`（step 1）成功、Campaign 在 Meta 那邊建立
- 失敗時 backend orphan cleanup 自動跑（campaign 在 Meta 已刪、DB 紀錄留 `status=failed` + `platform_campaign_id=None`）— **AI 不需手動 cleanup、不需 cancel**

❌ **AI 必說的事**：
> 「Meta 廣告目前後端有個 known bug、所有廣告組合會在 step 2 失敗（subcode 1870227）。已回報、修復前這個功能暫時不能用。如果你急著投廣告、可以用 `generate_ad` 生圖、然後手動到 Meta Ads Manager 自己建 campaign。」

❌ **AI 不要做的事**：
- 不要鼓勵用戶「再試一次看看」— 確定會 fail、是浪費時間
- 不要嘗試手動繞過（例如自己加 `advantage_audience` 欄位） — schema `extra=forbid`、會 422
- 不要嘗試 cleanup orphan — backend 已做、再做會 double delete

**Rule: TW 受眾單獨需要 advertiser verification、是用戶端設定問題（非 plugin bug）。**
即使 advantage_audience bug 修了、`target_countries: ["TW"]` 仍會卡 subcode 3858495、要求用戶到 Meta Business Manager 完成 advertiser verification（連結：https://www.facebook.com/business/help/983527276402621）。在那個流程跑完前、避免推薦 TW 投放、改用 US/JP/KR 等不需驗證的市場。

---

**⚠️ 參數名照 `AdCampaignCreateRequest` schema 原樣抄（順序不重要，名字錯就 422）：**

```
mcp_tool_call("zereo_social_mcp", "create_ad_campaign", {
  "user_token": token,
  "data_json": "{
    \"social_account_id\": \"<from list_accounts>\",
    \"campaign_name\": \"\",
    \"campaign_objective\": \"OUTCOME_TRAFFIC\",
    \"ad_type\": \"image\",
    \"creative_image_url\": \"<from generate_ad image_url>\",
    \"creative_message\": \"探索自然無毒的保養體驗 — 今天就試試\",
    \"cta_type\": \"SHOP_NOW\",
    \"cta_url\": \"https://landingai.info/zh-TW/lp/<campaign_id>\",
    \"daily_budget\": 5.0,
    \"target_age_min\": 25,
    \"target_age_max\": 40,
    \"target_genders\": [0],
    \"target_countries\": [\"TW\"],
    \"placements\": [\"facebook\", \"instagram\"],
    \"schedule_start\": \"2026-04-20T00:00:00+08:00\",
    \"schedule_end\": \"2026-05-05T00:00:00+08:00\"
  }"
})
→ Returns: AdCampaignResponse { id, status, platform_campaign_id, platform_ad_id, ... }
```

### 必填 vs 選填 + 型別/值域

| 欄位 | 必填 | 型別 | 預設/值域 | 說明 |
|------|:---:|------|----------|------|
| `social_account_id` | ✓ | string | — | `list_accounts` 回傳的 account id（**不是 `account_id`**） |
| `campaign_name` | — | string | 空字串會自動產生 | |
| `campaign_objective` | — | string | `"OUTCOME_TRAFFIC"` | `OUTCOME_AWARENESS / OUTCOME_TRAFFIC / OUTCOME_ENGAGEMENT / OUTCOME_LEADS / OUTCOME_SALES`（**注意前綴 `OUTCOME_`，不是 `TRAFFIC`、`CONVERSIONS`**） |
| `ad_type` | — | string | `"image"` | `image / video` |
| `creative_image_url` | ✓ 若 ad_type=image | string | — | 從 `generate_ad` 的 `image_url` 拿（**不是 `creative_id`**） |
| `creative_video_url` | ✓ 若 ad_type=video | string | — | |
| `creative_message` | — | string | 空字串 | 廣告文字內容 |
| `cta_type` | — | string | `"LEARN_MORE"` | `LEARN_MORE / SHOP_NOW / SIGN_UP / BOOK_TRAVEL / DOWNLOAD / CONTACT_US` 等 |
| `cta_url` | — | string | 空字串 | 廣告點擊去的 landing URL（**不是 `landing_url`**） |
| `daily_budget` | — | float | `5.0` | USD，最低 $1 |
| `target_age_min` | — | int | `18` | flat 欄位（**不是 `targeting.age_min`**） |
| `target_age_max` | — | int | `65` | |
| `target_genders` | — | list[int] | `[0]` | `[0]=all` / `[1]=male` / `[2]=female`（**是數字不是字串**） |
| `target_countries` | — | list[string] | `["TW"]` | ISO country code |
| `placements` | — | list[string] | `["facebook","instagram"]` | |
| `schedule_start` | — | ISO datetime | null | null = 立即開始 |
| `schedule_end` | — | ISO datetime | null | null = 無結束時間 |

**⚠️ schema 目前不支援 `interests`**（興趣定位）。若要 interest targeting 需另用 `update_campaign_targeting`（若存在）。

**objective → optimization_goal 後端自動 derive**（你不用傳 optimization_goal，backend 會根據 `campaign_objective` 配對正確的 optimization_goal）：
| `campaign_objective` | 自動 optimization_goal |
|---------------------|----------------------|
| `OUTCOME_AWARENESS` | `REACH` |
| `OUTCOME_TRAFFIC` | `LINK_CLICKS` |
| `OUTCOME_ENGAGEMENT` | `POST_ENGAGEMENT` |
| `OUTCOME_LEADS` | `LEAD_GENERATION` |
| `OUTCOME_SALES` | `OFFSITE_CONVERSIONS` |

## Phase 5b (TikTok branch): Create TikTok Ad Campaign

當使用者選擇 TikTok 投放、走這條（**不要** call Meta `create_ad_campaign`、schema 完全不同）。

### 🔴 TikTok 也要逐項確認 13 欄位（紀律同 Phase 4）

跟 Meta 一樣、不准 silent default、每欄位走「使用者親答」or「LLM 推 + 明說」。差別在 TikTok 的值域跟 Meta 不對稱（objective 沒 OUTCOME_ 前綴、年齡用離散桶、location 是數字 ID），別把 Meta 的值複製過來。

### 先查 TikTok-specific objectives + CTA
```
mcp_tool_call("zereo_social_mcp", "get_tiktok_ad_objectives", { "user_token": token })
→ REACH / TRAFFIC / VIDEO_VIEWS / ENGAGEMENT / LEAD_GENERATION / CONVERSIONS / PRODUCT_SALES / APP_PROMOTION
mcp_tool_call("zereo_social_mcp", "get_tiktok_ad_cta_types", { "user_token": token })
```

⚠️ **TikTok objectives 不是 Meta 的 `OUTCOME_*` 命名**——用 `TRAFFIC` 不是 `OUTCOME_TRAFFIC`。passing OUTCOME_* 會 422。

### TikTok 13 欄位逐項收集（4 批、跟 Meta 同節奏）

#### 批 1 — 帳號 + 素材
| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 1 | `social_account_id` | Phase 1 列表挑、**必須是 platform="tiktok" 的帳號**（用 Meta 的回 400） |
| 2 | `advertiser_id` | 空字串會 fallback 到 social_account.ad_account_id；若 Business Center 沒綁、跳出來請使用者貼 advertiser_id |
| 3 | `ad_type` + `creative_video_url` / `creative_image_url` | TikTok 強推 video（`ad_type="video"` 為預設）。建議從 `generate_reels` 拿 video，或 `generate_ad` 出 9:16 直版 image |

#### 批 2 — 廣告目標 + CTA
| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 4 | `campaign_objective` | **問**——8 個 TikTok 值（REACH / TRAFFIC / **VIDEO_VIEWS** / ENGAGEMENT / LEAD_GENERATION / CONVERSIONS / PRODUCT_SALES / APP_PROMOTION）。Meta 沒 `VIDEO_VIEWS`、TikTok 是常見選擇 |
| 5 | `cta_type` | TikTok CTA 比 Meta 多 2 個：`ORDER_NOW` / `SUBSCRIBE` / `INSTALL_NOW` / `WATCH_NOW`（共 10 個） |
| 6 | `cta_url` | 同 Meta、可拼 UTM |
| 7 | `creative_message` | TikTok 廣告文（max 1024 chars）、**問使用者**或從 `social_copy` 帶過來 |

#### 批 3 — 預算 + 排程
| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 8 | `daily_budget` | **問**——USD、TikTok 建議 **≥ $20/day**（高於 Meta 的 $5）。低於 $20 雖能投但 TikTok 演算法投不準 |
| 9 | `schedule_start` | **問**——ISO datetime、null = 立即。**注意**：TikTok 的 `schedule_type` 規則是「start 跟 end 必須同時有、否則一律 SCHEDULE_FROM_NOW」（已在 backend 處理、但提醒使用者就排程上的差別）|
| 10 | `schedule_end` | **問**——同上 |

#### 批 4 — 受眾定位
| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 11 | `target_age_groups` | **問**——TikTok 用**離散年齡桶**：`AGE_13_17` / `AGE_18_24` / `AGE_25_34` / `AGE_35_44` / `AGE_45_54` / `AGE_55_100`。可選多個。**禁止**直接複製 Meta 的 `target_age_min/max` 整數（schema `extra=forbid`、會 422）|
| 12 | `target_genders` | 同 Meta：[0]=all / [1]=male / [2]=female（後端會翻成 TikTok 的 `GENDER_UNLIMITED` / `GENDER_MALE` / `GENDER_FEMALE`）|
| 13 | `target_locations` | **問**——TikTok 用**數字 location_id**（158=Taiwan、1=USA、81=Japan、410=Korea、158 = TW、702 = SG、764 = TH...），**不是** ISO country code。給使用者人話選、然後 LLM 自己對應到 ID |
|  | `placements` | 預設 `["PLACEMENT_TIKTOK"]`。可加 `PLACEMENT_PANGLE`（TikTok 的廣告聯播網、第三方 app）但通常不建議 |

### Cost 複誦 + 啟動詞（同 Meta 紀律）

跑完 4 批整理複誦表：
```
TikTok 廣告投放規格：

帳號：@martin_tiktok（advertiser_id=...）
素材：video, https://...generate_reels.mp4
目標：TRAFFIC
CTA：LEARN_MORE → https://landingai.info/zh-TW/lp/<id>?utm_source=tiktok&utm_campaign=launch
廣告文：「[800 字以內]」

預算：$25 USD/day × 14 天 = $350 total
排程：2026-04-26 ~ 2026-05-10
受眾：年齡桶 [AGE_25_34, AGE_35_44]（從 Endurance Athletes TA 推、要改直接講）
      性別：[0] 不分（推）
      地區：[158]=Taiwan, [81]=Japan（你親選的）
      投放：PLACEMENT_TIKTOK

確認要建立嗎？
```

### Create TikTok campaign
```
mcp_tool_call("zereo_social_mcp", "create_tiktok_ad_campaign", {
  "user_token": token,
  "data_json": "{
    \"social_account_id\": \"<TikTok account id from list_accounts>\",
    \"campaign_objective\": \"TRAFFIC\",
    \"ad_type\": \"video\",
    \"creative_video_url\": \"<from generate_reels / external video URL>\",
    \"creative_message\": \"探索自然無毒的保養體驗 — 今天就試試\",
    \"cta_type\": \"SHOP_NOW\",
    \"cta_url\": \"https://landingai.info/zh-TW/lp/<campaign_id>\",
    \"daily_budget\": 20.0,
    \"target_age_groups\": [\"AGE_25_34\", \"AGE_35_44\"],
    \"target_genders\": [0],
    \"target_locations\": [\"158\"],
    \"placements\": [\"PLACEMENT_TIKTOK\"]
  }"
})
```

### 必填 vs 選填（TikTok schema、跟 Meta 不同）

| 欄位 | 必填 | 預設/值域 | 跟 Meta 的差異 |
|------|:---:|----------|----------------|
| `social_account_id` | ✓ | — | 必須是 TikTok 帳號（platform="tiktok"），用 Meta 帳號回 400 |
| `advertiser_id` | — | 空字串 → fallback 到 social_account.ad_account_id | TikTok-specific |
| `campaign_objective` | — | `"TRAFFIC"` | **不加 `OUTCOME_` 前綴**——bare 名稱 |
| `ad_type` | — | `"video"`（TikTok video-first）| Meta 預設 `image` |
| `creative_video_url` | ✓ if video | — | TikTok 推薦 video |
| `daily_budget` | — | `20.0` USD | TikTok 建議 ≥ $20/day（Meta 是 $5）|
| `target_age_groups` | — | `["AGE_25_34","AGE_35_44"]` | **離散桶**：AGE_13_17 / 18_24 / 25_34 / 35_44 / 45_54 / 55_100。**不是** age_min/age_max 整數 |
| `target_locations` | — | `["158"]` | **TikTok 數字 location_id**（158=Taiwan、1=USA）。**不是** ISO country code |
| `target_genders` | — | `[0]` | 同 Meta：[0]=all / [1]=male / [2]=female |
| `placements` | — | `["PLACEMENT_TIKTOK"]` | TikTok-specific 列舉 |
| `cta_type` | — | `"LEARN_MORE"` | LEARN_MORE / SHOP_NOW / SIGN_UP / DOWNLOAD / BOOK_NOW / CONTACT_US / ORDER_NOW / SUBSCRIBE / INSTALL_NOW / WATCH_NOW |

### 503 NOT_CONFIGURED 處理
若回 503 + `NOT_CONFIGURED`，server 缺 `TIKTOK_BUSINESS_APP_ID` env var。用人話跟使用者講：
> 「TikTok 廣告投放尚未在這個帳號的伺服器啟用、暫時無法跑。Meta 廣告可以照常走、或者我幫你生 TikTok 廣告素材、你自己上 TikTok Ads Manager 投。」

不要 retry、不要報技術細節。

### Pause / Resume
```
mcp_tool_call("zereo_social_mcp", "pause_tiktok_ad_campaign", {
  "user_token": token, "campaign_id": "<ZereoAdCampaign id>"
})
mcp_tool_call("zereo_social_mcp", "resume_tiktok_ad_campaign", {
  "user_token": token, "campaign_id": "<ZereoAdCampaign id>"
})
```

⚠️ `campaign_id` 是我們 DB 的 ZereoAdCampaign id、不是 TikTok 平台的 platform_campaign_id。

---

## Phase 6: Monitor

```
mcp_tool_call("zereo_social_mcp", "get_ad_campaign", {
  "user_token": token,
  "campaign_id": ad_campaign_id,
  "refresh": true
})
→ Returns: { "status": "active", "spent": 150.00, "impressions": 25000, "clicks": 750, "ctr": 3.0 }
```

### Campaign management
```
mcp_tool_call("zereo_social_mcp", "pause_ad_campaign", { "user_token": token, "campaign_id": id })
mcp_tool_call("zereo_social_mcp", "resume_ad_campaign", { "user_token": token, "campaign_id": id })
```

## Output

```
✅ Ad Campaign Created!

Platform: Meta (Facebook + Instagram)
Objective: Traffic
Budget: $50/day × 15 days = $750 total
Status: Pending Review (Meta typically approves within 24h)
Campaign ID: [id]

Targeting:
- Age 25-40, Taiwan
- Interests: Skincare, Beauty

Creative: Variant A — "Discover Premium Skincare"
Landing URL: [url]

Next steps:
A) Monitor campaign performance
B) Export creative for manual Google Ads upload (if user wants Google Ads — plugin cannot create Google Ads campaigns, user uploads via Google Ads Manager)
C) Done
```

## Reel Promotion (Instagram/Facebook Boost)

`promote_reel` 是 boost **已發佈的 IG Reel** 的捷徑（比 `create_ad_campaign` 簡單、不用自己挑 image / video URL、直接用發佈過的 Reel）。

### 🔴 8 個欄位逐項確認

跟完整 ad campaign 一樣紀律、不准 silent default：

| # | 欄位 | 怎麼決定 |
|---|------|---------|
| 1 | `social_post_id` | 從 `get_post_history(status_filter="published")` 列出 IG Reel 讓使用者挑（**只能 boost 已發佈的 ig_reel、不能是 ig_post / ig_story / fb_post**）|
| 2 | `cta_type` | 8 個值（LEARN_MORE / SHOP_NOW / SIGN_UP / BOOK_TRAVEL / DOWNLOAD / 等）— **問** |
| 3 | `cta_url` | **問**——點 CTA 去哪、可拼 UTM |
| 4 | `daily_budget` | **問**——USD、最低 $1。建議 $5-30 |
| 5 | `target_age_min` | 從 TA 推 + 明說、預設 18 |
| 6 | `target_age_max` | 同上、預設 65 |
| 7 | `target_countries` | **問**——ISO code 陣列、預設 ["TW"]（注意 TW 要 advertiser verification）|
| 8 | `caption` | **問**——Boost 時可改 ad caption（覆蓋原 Reel）；不傳 = 用原 caption |

### Cost 複誦 + 啟動詞
```
要 boost 哪一個 Reel？

A) 「FocusFit 30hr 續航實測」 (post_id=...) — 5,234 views, posted 4-20
B) 「鐵人三項耳機推薦」 (post_id=...) — 1,892 views, posted 4-22

選好之後：
- CTA：____ → ____
- 預算：$__ USD/day
- 受眾：年齡 __-__、國家 [__]
- Caption：[沿用原 caption / 改 ___]
```

接受啟動詞「開始 / boost / go」才送 `promote_reel`。

### Promote a published reel/post
```
mcp_tool_call("zereo_social_mcp", "promote_reel", {
  "user_token": token,
  "data_json": "{\"social_post_id\": \"<post_id_from_get_post_history>\", \"cta_type\": \"LEARN_MORE\", \"cta_url\": \"https://landing-page-url\", \"daily_budget\": 10.00, \"target_age_min\": 25, \"target_age_max\": 45, \"target_countries\": [\"TW\"], \"caption\": \"\"}"
})
→ Returns: { "promotion_id": "..." }
```

### Check promotion status
```
mcp_tool_call("zereo_social_mcp", "get_promotion_status", {
  "user_token": token, "promotion_id": "<promotion_id>"
})
```

### Pause / Resume promotion
```
mcp_tool_call("zereo_social_mcp", "pause_promotion", {
  "user_token": token, "promotion_id": "<promotion_id>"
})

mcp_tool_call("zereo_social_mcp", "resume_promotion", {
  "user_token": token, "promotion_id": "<promotion_id>"
})
```

## Multi-Platform Campaigns

To run on both Meta and Google:
1. Create Meta campaign (steps above)
2. Generate Google-specific creative: `generate_ad(session_id, platform="google")`
3. Create Google campaign with responsive display/search ad format
4. Monitor both campaigns

Present unified dashboard:
```
Active Campaigns:
━━━━━━━━━━━━━━━━━
📘 Meta — $50/day — Active — CTR: 3.0%
🔍 Google — $40/day — Active — CTR: 2.1%
━━━━━━━━━━━━━━━━━
Total daily spend: $90
```

## SaleCraft Scope & Pricing (MUST READ)

### Who We Serve
SaleCraft is for **physical product sellers only** (skincare, food, fashion, health, electronics, etc.).
- ✅ Physical products, single items, single purpose
- ❌ Software/SaaS, multi-purpose platforms, abstract services

If the user's product doesn't fit, politely redirect:
> "SaleCraft 主要服務實體產品的行銷。你的需求可能更適合其他方案。"

### Pricing — Tell Before You Act
**1 USD = 1 pt | Minimum top-up: $20 = 20 pts | USD only — never NT$ / EUR / £ / ¥ / 円 / 人民幣 / KRW / THB / VND / 任何其他幣別（see `lib/credit-calculator.md` § Currency Rule）**

This skill's cost depends on the ad creative generation: quick ad image = 200 pts (~$7), carousel = 300 + 100×N pts. Campaign creation itself uses the generated creative.

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

### Before ad creation:
```
準備投放廣告！

1. ✅ 確認廣告帳號（Meta / Google）
2. 🎯 選擇投放目標（曝光 / 流量 / 互動 / 名單 / 銷售）
3. 🎨 AI 生成廣告素材
4. 💰 設定每日預算
5. 📊 查看現有廣告表現
```

### After campaign created:
```
✅ 廣告活動已建立！

狀態：等待平台審核（通常 24 小時內）

接下來：
1. 📊 監控廣告表現
2. ⏸ 暫停 / 恢復廣告
3. 🎨 建立另一個平台的廣告（Meta → Google）
4. 📱 回到社群發佈
5. 🏠 回到首頁建置
```
