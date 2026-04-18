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
1. 📘 Meta Ads — ACME Ad Account (can create campaigns)
2. 🔍 Google Ads — ACME Search (can create campaigns)

Select platform(s) for ad campaign:
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

## Phase 4: Set Budget & Targeting

### Budget
```
Set your daily budget:

Platform: Meta
Objective: Traffic
Recommended: $30-100/day for meaningful data

Your daily budget (USD): ___
Campaign duration: [start_date] to [end_date]
```

### Targeting
Audience targeting from Phase 2 (audience-target) carries forward:
- Demographics (age, gender, location)
- Interests (from TA profile)
- Custom audiences (if available)

```
Targeting Summary:
- Age: 25-40
- Gender: All
- Location: Taiwan
- Interests: Skincare, Beauty, Health
- Language: zh-TW

Adjust targeting or proceed?
```

## Phase 5: Create Campaign

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
A) Create Google Ads campaign too
B) Monitor campaign performance
C) Done
```

## Reel Promotion (Instagram/Facebook Boost)

### Promote a published reel/post
```
mcp_tool_call("zereo_social_mcp", "promote_reel", {
  "user_token": token,
  "data_json": "{\"social_post_id\": \"<post_id_from_get_post_history>\", \"cta_type\": \"LEARN_MORE\", \"cta_url\": \"https://landing-page-url\", \"daily_budget\": 10.00, \"duration_days\": 7}"
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
**1 USD = 30 pts | Minimum top-up: $20 = 600 pts**

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
