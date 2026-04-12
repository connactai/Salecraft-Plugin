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

If user has no connected accounts, guide them through OAuth:

### Connect Meta (Facebook + Instagram)
```
mcp_tool_call("zereo_social_mcp", "get_meta_auth_url", {
  "user_token": token,
  "redirect_uri": "https://landingai.info/auth/callback"
})
→ Returns: { "auth_url": "https://www.facebook.com/v20.0/dialog/oauth?..." }
```
Give the user the `auth_url` to open in browser. After they authorize, Meta redirects
with a `code` parameter. Then complete the connection:
```
mcp_tool_call("zereo_social_mcp", "connect_meta_account", {
  "user_token": token,
  "code": "<code_from_redirect>",
  "redirect_uri": "https://landingai.info/auth/callback"
})
→ Returns: newly connected account details (Facebook page + Instagram business)
```

### Connect TikTok
```
mcp_tool_call("zereo_social_mcp", "get_tiktok_auth_url", {
  "user_token": token
})
→ Returns: { "auth_url": "https://www.tiktok.com/v2/auth/authorize/?..." }
```
TikTok callback is handled server-side — user just needs to open the URL and authorize.

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
