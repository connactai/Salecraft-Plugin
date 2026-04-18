---
name: publish-social
description: |
  Publish landing page content to social media platforms. Lists connected accounts,
  imports LP content, validates media specs, suggests optimal timing, and publishes
  to multiple platforms simultaneously via zereo_social_mcp.
  Trigger: Phase 6a of /salecraft-publish, or "post to social media", "publish to Instagram".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Social Publishing — Multi-Platform Content Distribution

## 🚨 STOP — READ THIS FIRST

**This skill EXECUTES via API. It does NOT write a caption + post-it-yourself instructions.**

When the user says "幫我發到 IG / 發到 FB / publish to social / post this": per CLAUDE.md FIRST-RESPONSE RULE, your first reply contains ONLY (1) "this is paid (~100 pts/set)" + Meta-account-binding requirement (Pro/Business account on marketingx page), (2) AI Token 3-step prompt, (3) optional 1-line scope question. NO drafted caption, NO hashtag list, NO "here's what I'd post" content in the first reply.

After token + connected account → call `publish_post` / `POST /api/social/publish`. Return the real post URL. **Never** end with "now you can copy this caption and post it manually" — that's the user doing your job.

---

You manage social media publishing for LP content. You help the user push their landing page visuals and copy to connected social platforms **by calling the publish API** — not by drafting caption text and asking the user to post it themselves.

## Prerequisites

- `user_token` and `session_id` from previous phases
- Read `CLAUDE.md` for zereo_social_mcp tool signatures + the FIRST-RESPONSE RULE at the top

## Phase 0: Connect Social Accounts (if not connected)

If user has no connected accounts, **direct them to the SaleCraft frontend** — do NOT generate OAuth URLs yourself.

### Connect Meta (Facebook + Instagram)

**⚠️ IMPORTANT:**
1. IG 必須是**專業帳戶（Professional/Business）**，個人帳戶無法透過 API 發文
2. **不要**自己呼叫 `get_meta_auth_url` — OAuth redirect 只設定在前端

**正確步驟**：
> 「請到這個頁面連結你的 Meta 帳號：
> https://salecraft.ai/{locale}/marketingx
>
> 注意：你的 IG 帳號必須是**專業帳戶**或**商業帳戶**，個人帳戶無法透過 API 發文。
> 如果還不是，到 IG App → 設定 → 帳號 → 切換為專業帳號。
>
> 連結完成後，回到對話複製頁面上的「**AI 登入 Token**」貼給我，我就能直接幫你發文。」

連結完成後，用 `list_accounts(user_token)` 確認是否已連結。

### Connect TikTok

**⚠️ 重要：**
1. **不要**自己呼叫 `get_tiktok_auth_url` — OAuth redirect 只設定在前端
2. TikTok 目前在**沙盒模式（sandbox）**，有幾個行為會讓你（AI）跟用戶看起來像 bug，其實是正常的 — 看下面「TikTok Publishing」章節

**正確步驟**：
> 「請到這個頁面連結你的 TikTok 帳號：
> https://salecraft.ai/{locale}/marketingx
>
> 點 STEP 4 的「連結 TikTok」按鈕，授權後會自動跳回頁面，TikTok 帳號卡片會變「已完成」。
>
> 連結完成後，回到對話複製頁面上的「**AI 登入 Token**」貼給我，我就能直接幫你發影片。」

連結完成後，用 `list_accounts(user_token)` 確認是否已連結。

**TikTok 帳號綁好後，你會在 `list_accounts` 看到類似這樣的資料**（`account_capability` 的欄位很重要）：
```
{
  "id": "aaf28e61-...",
  "platform": "tiktok",
  "account_name": "Martin",
  "can_publish": false,         ← 沙盒帳號永遠 false，不代表不能發（見下方說明）
  "account_capability": {
    "privacy_options": ["FOLLOWER_OF_CREATOR", "MUTUAL_FOLLOW_FRIENDS", "SELF_ONLY"],
    "can_post_public": false,   ← 沙盒 = false
    "max_video_duration": 3600, ← 秒；沙盒帳號通常 3600s（60min）
    "duet_disabled": true,
    "stitch_disabled": true,
    "comment_disabled": false
  }
}
```

## TikTok Publishing — 沙盒模式 + 完整參數

### 你（AI）必須知道的 3 條沙盒事實

**⚠️ 在回答用戶任何 TikTok 相關問題前，先把這 3 條讀進去。**

1. **沙盒強制 `SELF_ONLY`**
   - TikTok 在 App Review 通過前，不管用戶傳什麼 `privacy_level`，後端會自動降級成 `SELF_ONLY`（僅自己可見）
   - 影片確實會發上去，但只有用戶自己在 TikTok App 的「個人檔案 → 僅自己可見」分類看得到
   - **不要**跟用戶說「發文失敗」— 它成功了，只是發成私人
   - 用戶問「為什麼別人看不到？」→ 回答：「目前 TikTok 是沙盒模式，發上去的影片預設只有你自己看得到。要公開發文需要 TikTok 審核通過整個 App（這是 TikTok 那邊的流程，不是我們這邊）。」

2. **`platform_permalink` 會是空字串**
   - 成功發文後 `platform_permalink: ""` 是正常的，不是 bug
   - TikTok 沙盒 API 不會回傳公開 post URL（因為影片本來就不公開）
   - **不要**試圖組一個 TikTok URL 給用戶 — 就說「影片已上傳到你的 TikTok 個人檔案，打開 TikTok App 可以看到」

3. **`can_publish: false` 不代表不能發**
   - 這個欄位是給前端 UI 顯示用的「能否發公開」提示
   - 沙盒帳號永遠 `false`，但你呼叫 `publish_post` 一樣會成功（backend 自動降級）
   - **不要**看到 `can_publish: false` 就拒絕幫用戶發文

### tt_video 發片標準流程

```
# 1. 用 list_accounts 找到 tiktok 帳號的 id
mcp_tool_call("zereo_social_mcp", "list_accounts", { "user_token": token })
→ 從回傳中找 platform == "tiktok" 的那筆，記下它的 "id"

# 2. 準備影片 URL（公開可下載的 https URL）
# 可以從以下來源：
#   - 用戶給的公開影片 URL
#   - Reels 生成完成後的 final_video_url（landing_ai_mcp get_reel_result）
#   - GCS signed URL（至少 1 小時有效）

# 3. 發文
mcp_tool_call("zereo_social_mcp", "publish_post", {
  "user_token": token,
  "data_json": "{
    \"social_account_id\": \"<tiktok_account_id>\",
    \"post_type\": \"tt_video\",
    \"caption\": \"影片標題（TikTok 限制 150 字）#hashtag1 #hashtag2\",
    \"video_url\": \"https://...your-video.mp4\",
    \"privacy_level\": \"PUBLIC_TO_EVERYONE\",
    \"disable_duet\": false,
    \"disable_stitch\": false,
    \"disable_comment\": false
  }"
})
→ { "id": "<post_id>", "status": "draft", ... }

# 4. 輪詢狀態（immediate publish，通常 10-30 秒內完成）
mcp_tool_call("zereo_social_mcp", "get_post_detail", {
  "user_token": token,
  "post_id": "<post_id>"
})
→ status: "published" | "failed" | "publishing"
```

### tt_video 參數完整表

| 參數 | 型別 | 必填 | 說明 |
|------|------|------|------|
| `social_account_id` | string | ✓ | `list_accounts` 回傳的 tiktok 帳號 id |
| `post_type` | string | ✓ | 固定 `"tt_video"` |
| `video_url` | string | ✓ | 公開可下載的 mp4 URL（建議 9:16、最大 500MB） |
| `caption` | string | — | 標題，最多 150 字，hashtag 包含在內 |
| `privacy_level` | string | — | `PUBLIC_TO_EVERYONE` / `MUTUAL_FOLLOW_FRIENDS` / `FOLLOWER_OF_CREATOR` / `SELF_ONLY`（沙盒強制 SELF_ONLY） |
| `disable_duet` | bool | — | 關閉 duet 功能（default false） |
| `disable_stitch` | bool | — | 關閉 stitch 功能（default false） |
| `disable_comment` | bool | — | 關閉留言（default false） |
| `scheduled_at` | ISO datetime | — | 排程時間；不傳即立即發 |

### 影片規格要求

- **比例**：9:16 最佳（vertical），支援 1:1 / 16:9 但會被 TikTok 裁切
- **長度**：沙盒帳號最多 60 分鐘（`max_video_duration: 3600s`），一般帳號最多 3 分鐘
- **檔案大小**：最大 500MB
- **格式**：MP4（H.264 + AAC）

### tt_photo（TikTok 圖文貼文）

```
mcp_tool_call("zereo_social_mcp", "publish_post", {
  "user_token": token,
  "data_json": "{
    \"social_account_id\": \"<tiktok_account_id>\",
    \"post_type\": \"tt_photo\",
    \"caption\": \"圖文標題\",
    \"image_url\": \"https://...verified-domain.com/photo.jpg\",
    \"privacy_level\": \"PUBLIC_TO_EVERYONE\"
  }"
})
```

**⚠️ tt_photo 限制：**
- 只支援 `PULL_FROM_URL`（TikTok 下載 URL）— image URL 的 domain 必須在 TikTok Developer Portal 驗證過，或使用 GCS signed URL（從已驗證 domain 發出）
- 格式：**只支援 JPG / JPEG / WEBP**，**不支援 PNG**
- 最多 35 張（走 `image_urls` array）
- 每張最大 20MB

若用戶給的是 PNG 或 domain 未驗證的 URL，先用 `upload_base64` 把圖轉存到 GCS 再發。

### 常見錯誤 + 應對

| `error_message` 關鍵字 | 意義 | 該怎麼跟用戶說 |
|----------------------|------|---------------|
| `integration guidelines` | 沙盒 + privacy_level 不合法 | 你（AI）不用處理 — backend 會自動降級成 SELF_ONLY 重試；如果最終還是失敗代表真的有別的問題 |
| `Token refresh failed` | TikTok token 過期且 refresh 失敗（~24h 失效） | 「你的 TikTok 授權已過期，請到 `https://salecraft.ai/{locale}/marketingx` 重新連結 TikTok 帳號。」 |
| `Failed to download video` | 我方抓不到用戶給的 video_url | 「影片網址下載失敗，請確認網址可以公開存取（不需要登入）且 1 小時內不會過期。」 |
| `Video upload failed: HTTP 4xx` | TikTok 拒絕影片檔（格式、大小、長度） | 檢查影片規格（見上方） |
| `Polling timed out` | TikTok 處理超過 2.5 分鐘還沒完成 | 「TikTok 這次處理比較慢，影片可能還在後台轉檔 — 請 5 分鐘後打開 TikTok App 看「僅自己可見」分類確認是否發出。」 |

### 升級到公開發文（App Review 路徑）

如果用戶抱怨「只能發私人很不方便」，給他這個路徑：

1. **短期（開發者 Target Users 名單）**：App 擁有者可以在 TikTok Developer Portal 加入最多 10 個「Target Users」的 TikTok 帳號，這些帳號可以繞過沙盒直接發公開
2. **長期（App Review）**：向 TikTok 送 App Review（需要隱私政策、資料使用說明、App 用途展示影片）。通過後所有用戶都能發公開

這個決定是 SaleCraft 團隊要做的，**不是用戶能自己處理的**。若用戶真的有公開發文需求，請他寄信到 `zereo@connact.ai` 說明用量，團隊會評估送審時程。

## Social Post Generation (Image + Caption)

**⚠️ 用戶說「發一則貼文」= 需要圖片 + 文案，不是純文字。**

### 快速生成廣告圖（推薦，~5 分鐘）
```
1. generate_ad(session_id, { platform: "meta", ta_group_id: "ta_1" })
   → project_id, status: "processing"
2. 每 30 秒 poll: get_ad_result(session_id, project_id)
   → status: "completed" → 取得 image_url
3. social_copy(user_token, { conversation_id, quantity: 1 })
   → 生成文案
4. publish_post({ social_account_id, post_type: "ig_post", caption, image_url })
```

### 從 Landing Page 取圖
```
1. download_stripe(campaign_id, stripe_idx) → image URL
2. social_copy → 文案
3. publish_post(...)
```

**時間：廣告圖 ~5 分鐘，LP 取圖 ~1 分鐘。不要跟用戶說要 30 分鐘。**

### Account Management

**Refresh expired token** (Meta tokens expire ~60 days):
```
mcp_tool_call("zereo_social_mcp", "refresh_account_token", {
  "user_token": token,
  "account_id": "<account_id_from_list_accounts>"
})
→ Returns: refreshed account with new token_expires_at
```
When to use: if `list_accounts` shows `token_expires_at` is near or past.

**Recheck account capabilities** (after changing FB page settings):
```
mcp_tool_call("zereo_social_mcp", "recheck_capability", {
  "user_token": token,
  "account_id": "<account_id>"
})
→ Triggers async check — call get_account_capability after to see result
```

**Update ad account** (link a different ad account ID):
```
mcp_tool_call("zereo_social_mcp", "update_ad_account", {
  "user_token": token,
  "account_id": "<account_id>",
  "data_json": "{\"ad_account_id\": \"<new_ad_account_id>\"}"
})
```

**Get account's published content** (what's already posted):
```
mcp_tool_call("zereo_social_mcp", "get_account_content", {
  "user_token": token,
  "account_id": "<account_id>"
})
→ Returns: list of posts published via this account
```

**Disconnect account** (⚠️ destructive — cannot undo):
```
mcp_tool_call("zereo_social_mcp", "disconnect_account", {
  "user_token": token,
  "account_id": "<account_id>"
})
→ Removes the social account connection. User must re-authorize to reconnect.
```
Always confirm with user before disconnecting.

## Phase 1: Check Connected Accounts

```
mcp_tool_call("zereo_social_mcp", "list_accounts", {
  "user_token": token
})
→ Returns: array of connected social accounts with platform, name, capabilities
```

Present:
```
Connected Accounts:
1. 📘 Facebook — ACME Official Page
2. 📸 Instagram — @acme_brand
3. 🎵 TikTok — @acme_official

Select platforms to publish to (e.g., "1 and 2"):
```

If no accounts connected → **go to Phase 0 to connect first.**

## Phase 1.5: Generate Social Post Copy (AI)

Before importing or composing manually, use AI to generate platform-optimized captions:

### Generate social media copy
```
mcp_tool_call("landing_ai_mcp", "social_copy", {
  "user_token": token,
  "data_json": "{\"conversation_id\": \"<optional>\", \"quantity\": 1}"
})
→ Deducts credits and returns social copy sets
```

### Get session-generated content (includes social copy from Architect)
```
mcp_tool_call("landing_ai_mcp", "get_session_content", {
  "user_token": token,
  "session_id": "<session_id>"
})
→ Returns: rendered content with stripes + social copy (caption, hashtags) from generation
```

### Reflect on post quality (AI scoring)
```
mcp_tool_call("landing_ai_mcp", "social_reflect", {
  "user_token": token,
  "data_json": "{\"post_text\": \"Your caption here\", \"image_url\": \"https://...\", \"brand_name\": \"Brand\"}"
})
→ Returns: { "reflection_score": 8.5, "reflection_data": { strengths, weaknesses, suggestions } }
```
Use this to check if the caption is good before publishing. Score < 7 = rewrite recommended.

### Workflow
1. `get_session_content` → check if Architect already generated social copy
2. If yes → use it directly
3. If no or want more → `social_copy` to generate new copies (costs credits)
4. `social_reflect` → score the copy before publishing
5. Iterate until score ≥ 7

## Phase 2: Import LP Content

```
mcp_tool_call("zereo_social_mcp", "import_from_session", {
  "user_token": token,
  "session_id": session_id
})
→ Imports LP stripe images and copy into the content library
```

## Phase 3: Validate Media

Check if the imported content meets platform requirements:
```
mcp_tool_call("zereo_social_mcp", "validate_content_media", {
  "user_token": token,
  "content_id": "<content_id_from_import>",
  "target_post_type": "ig_post",
  "duration_seconds": 0
})
→ Returns: validation results (size, format, aspect ratio checks)
```

**Valid `target_post_type` values:**
- `ig_post` — Instagram feed post (1:1 or 4:5)
- `ig_story` — Instagram story (9:16)
- `ig_reel` — Instagram reel (9:16 video)
- `fb_post` — Facebook page post
- `tt_video` — TikTok video (9:16) — **沙盒限制見上方「TikTok Publishing」章節**
- `tt_photo` — TikTok 圖文貼文（JPG/JPEG/WEBP，最多 35 張，**不支援 PNG**）

If validation fails: report issues and suggest fixes (resize, reformat).

## Phase 4: Compose Posts

Help user craft platform-specific captions:

```
Suggested captions:

📘 Facebook:
"Discover [product] — designed for [TA]. [Value proposition]. 
Learn more: [landing_page_url] #[hashtag1] #[hashtag2]"

📸 Instagram:
"✨ [Short hook]
[Key benefit 1]
[Key benefit 2]
[CTA] — Link in bio!
.
.
#[hashtag1] #[hashtag2] ... #[hashtag10]"

🎵 TikTok:
"[Trend-aware hook] #[hashtag] #fyp"

Edit any caption, or approve all?
```

## Phase 5: Schedule or Publish Now

### Get optimal times
```
mcp_tool_call("zereo_social_mcp", "suggest_schedule", {
  "user_token": token,
  "content_id": "<content_id>",
  "caption": "Your post caption here",
  "content_type": "image",
  "ta_info_json": ""
})
→ Returns: recommended posting times per platform based on audience activity
```

### Or use natural language scheduling
```
mcp_tool_call("zereo_social_mcp", "parse_schedule", {
  "user_token": token,
  "text": "明天晚上8點"
})
→ { "schedules": [{ "datetime_iso": "2026-04-13T20:00:00+08:00", "platforms": ["ig_post"] }] }
```

Present:
```
Best posting times:
- Facebook: Today 7:00 PM (highest engagement)
- Instagram: Today 8:30 PM
- TikTok: Today 9:00 PM

A) Publish now
B) Schedule at optimal times
C) Custom schedule
```

### Publish
```
mcp_tool_call("zereo_social_mcp", "publish_multi", {
  "user_token": token,
  "targets_json": "[
    {\"social_account_id\": \"fb_123\", \"caption\": \"...\", \"media_ids\": [\"...\"]},
    {\"social_account_id\": \"ig_456\", \"caption\": \"...\", \"media_ids\": [\"...\"]},
    {\"social_account_id\": \"tt_789\", \"caption\": \"...\", \"media_ids\": [\"...\"]}
  ]"
})
```

### Or use publish_post for single platform
```
mcp_tool_call("zereo_social_mcp", "publish_post", {
  "user_token": token,
  "data_json": "{\"social_account_id\": \"...\", \"post_type\": \"ig_post\", \"caption\": \"...\", \"image_url\": \"...\"}"
})
```
⚠️ Parameter name is `social_account_id` in ALL publish tools.

### Or schedule
```
mcp_tool_call("zereo_social_mcp", "schedule_post", {
  "user_token": token,
  "data_json": "{\"social_account_id\": \"...\", \"scheduled_at\": \"2026-04-15T19:00:00+08:00\", ...}"
})
```

## Phase 6: QR Code (Optional)

For print materials or physical displays:

```
mcp_tool_call("zereo_social_mcp", "generate_qr", {
  "user_token": token,
  "url": landing_page_url,
  "style": "branded"
})
→ Returns: QR code image URL
```

## Phase 7: Post-Publish Management

### View post history
```
mcp_tool_call("zereo_social_mcp", "get_post_history", {
  "user_token": token, "limit": 20, "offset": 0, "status_filter": ""
})
→ Returns: array of posts with status (published/scheduled/skipped), platform_permalink, metrics
```

### View single post detail
```
mcp_tool_call("zereo_social_mcp", "get_post_detail", {
  "user_token": token, "post_id": "<post_id>"
})
```

### Cancel scheduled post
```
mcp_tool_call("zereo_social_mcp", "cancel_post", {
  "user_token": token, "post_id": "<post_id>"
})
```

### Parse natural language schedule
```
mcp_tool_call("zereo_social_mcp", "parse_schedule", {
  "user_token": token, "text": "tomorrow at 8pm"
})
→ Returns: { "schedules": [{ "datetime_iso": "2026-04-13T20:00:00+08:00", "platforms": ["ig_post"] }] }
```

## Phase 8: Content Library Management

### List all content in library
```
mcp_tool_call("zereo_social_mcp", "list_content", { "user_token": token })
```

### Get content detail
```
mcp_tool_call("zereo_social_mcp", "get_content", {
  "user_token": token, "content_id": "<content_id>"
})
```

### Update content (caption, hashtags)
```
mcp_tool_call("zereo_social_mcp", "update_content", {
  "user_token": token, "content_id": "<content_id>",
  "data_json": "{\"caption\": \"New caption\", \"hashtags\": [\"#tag1\"]}"
})
```

### Delete content
```
mcp_tool_call("zereo_social_mcp", "delete_content", {
  "user_token": token, "content_id": "<content_id>"
})
```

### Publish content directly
```
mcp_tool_call("zereo_social_mcp", "publish_content", {
  "user_token": token, "content_id": "<content_id>",
  "data_json": "{\"social_account_id\": \"<account_id>\", \"post_type\": \"ig_post\"}"
})
```
⚠️ Parameter name is `social_account_id` (NOT `account_id`).

��️ **Aspect ratio matters for ig_post**: LP images (9:16 tall) will fail on `ig_post` (requires 1:1 or 4:5).
Use `ig_story` for 9:16 content, or use the `quick_ad` image (1:1) for `ig_post`.

### Upload to content library (signed URL)
```
mcp_tool_call("zereo_social_mcp", "get_upload_signed_url", {
  "user_token": token, "filename": "image.jpg", "content_type": "image/jpeg"
})
→ upload via curl → then confirm_upload
```

## Phase 9: QR Code Tools

### Simple QR code
```
mcp_tool_call("zereo_social_mcp", "generate_qr", {
  "user_token": token, "url": "https://...", "size": 300
})
```

### QR composite (overlay QR on an image)
```
mcp_tool_call("zereo_social_mcp", "composite_qr", {
  "user_token": token,
  "image_url": "<background_image_url>",
  "qr_target_url": "https://landing-page-url"
})
→ Returns QR overlaid on the image (bottom-right)
```

### QR card (branded card with QR + title)
```
mcp_tool_call("zereo_social_mcp", "generate_qr_card", {
  "user_token": token, "url": "https://...", "title": "Brand Name"
})
→ Returns styled QR card image
```

## Output

```
✅ Published successfully!

📘 Facebook — Posted at [time] — [post_url]
📸 Instagram — Scheduled for [time]
🎵 TikTok — Posted at [time] — [post_url]

Want to:
A) Run ad campaigns on these platforms → /salecraft-publish (ads)
B) Check post history & engagement
C) Generate QR code for print materials
D) Done
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

Social copy generation costs **100 pts per set** (~$3). Publishing itself is free; the cost is in content generation (social_copy = 100 pts, quick ad image = 200 pts).

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

### Before publishing:
```
準備發佈到社群媒體！

1. 📱 連接社群帳號（Meta / TikTok）
2. 📝 AI 生成貼文文案
3. 📷 從 LP 匯入素材
4. ⏰ 查看最佳發佈時間
5. 📊 查看過去的發佈記錄
```

### After publishing:
```
✅ 發佈成功！

📱 [平台] — [貼文連結]

接下來：
1. 📊 投放廣告 — 把這則貼文推廣出去
2. 📝 發佈到其他平台
3. ⏰ 排程更多貼文
4. 📈 查看貼文表現數據
5. 🔲 產生 QR Code — 印刷品或實體展示用
6. 🏠 回到首頁建置
```
