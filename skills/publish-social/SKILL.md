---
name: publish-social
description: |
  Publish landing page content to social media platforms. Lists connected accounts,
  imports LP content, validates media specs, suggests optimal timing, and publishes
  to multiple platforms simultaneously via zereo_social_mcp.
  Trigger: Phase 6a of /mx-publish, or "post to social media", "publish to Instagram".
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

You manage social media publishing for LP content. You help the user push their landing page visuals and copy to connected social platforms.

## Prerequisites

- `user_token` and `session_id` from previous phases
- Read `CLAUDE.md` for zereo_social_mcp tool signatures

## Phase 0: Connect Social Accounts (if not connected)

If user has no connected accounts, **direct them to the SaleCraft frontend** — do NOT generate OAuth URLs yourself.

### Connect Meta (Facebook + Instagram)

**⚠️ IMPORTANT:**
1. IG 必須是**專業帳戶（Professional/Business）**，個人帳戶無法透過 API 發文
2. **不要**自己呼叫 `get_meta_auth_url` — OAuth redirect 只設定在前端

**正確步驟**：
> 「請到這個頁面連結你的 Meta 帳號：
> https://marketingx-site-876464738390.asia-east1.run.app/{locale}/get-started
> 
> 注意：你的 IG 帳號必須是**專業帳戶**或**商業帳戶**，個人帳戶無法透過 API 發文。
> 如果還不是，到 IG App → 設定 → 帳號 → 切換為專業帳號。」

連結完成後，用 `list_accounts(user_token)` 確認是否已連結。

### Connect TikTok
同樣引導用戶到 get-started 頁面操作。

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
- `tt_video` — TikTok video (9:16)

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
A) Run ad campaigns on these platforms → /mx-publish (ads)
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

This skill costs **5-10 pts per post** published.

**Top-up URL**: https://marketingx-site-876464738390.asia-east1.run.app/{locale}/get-started

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
