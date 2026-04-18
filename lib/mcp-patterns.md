# MCP Call Patterns Reference

> **No MCP in your runtime?** Skip this file and read [`rest-api-direct.md`](./rest-api-direct.md) instead — every MCP tool below has a 1:1 REST counterpart on the SaleCraft backend (currently `https://marketing-backend-v2-s6ykq3ylca-de.a.run.app`). Same auth, same payloads.

## Universal Call Pattern

All MCP tools are called through the Service System Deep Research proxy:

```
mcp__claude_ai_Service_System_Deep_Research__mcp_tool_call(
  server_name = "<mcp_server_name>",
  tool_name   = "<tool_name>",
  arguments   = { "param1": "value1", "param2": "value2" }
)
```

## Authentication Flow (AI Token only — never email/password)

### Step 1: User generates an AI Token

You (the AI) tell the user (with `{locale}` replaced for their language):

> 「① 開這個連結登入：https://salecraft.ai/{locale}/marketingx
> ② 點頁面上的「複製 AI 登入 Token」按鈕
> ③ 把 `sc_live_…` 貼回來給我」

### Step 2: Exchange the AI Token for an access_token
```
mcp_tool_call(
  server_name = "landing_ai_mcp",
  tool_name   = "authenticate_with_token",
  arguments   = { "ai_token": "sc_live_..." }
)
→ Returns: { "access_token": "eyJ...", "token_type": "bearer", "scope": "ai_agent" }
```

### Step 3: Use the access_token
Every subsequent call includes `user_token = access_token`:
```
mcp_tool_call(
  server_name = "landing_ai_mcp",
  tool_name   = "list_brands",
  arguments   = { "user_token": "eyJ..." }
)
```

### Step 4: On 401 (token expired or revoked)
```
# NOTE: authenticate_with_token does NOT return a refresh_token.
# When you get 401, ask the user to re-copy a fresh AI Token from
# https://salecraft.ai/{locale}/marketingx and call authenticate_with_token again.
# NEVER fall back to email/password — those tools are deprecated for AI use.
# Token expires in ~12 hours.
```

### Forbidden tools (deprecated for AI use)
Do **not** call any of these — they require credentials that the AI must never handle:
- `login(email, password)`
- `register(email, password, full_name)`
- `forgot_password(email)`
- `reset_password(token, new_password)`
- `verify_email(token)`
- `resend_verification(email)`

Account creation, password reset, and email verification all happen on the marketingx page itself. The user does it, then comes back with an AI Token.

## Common Patterns

### Pattern: Create-and-Poll (for generation)
```python
# 1. Create
session = mcp_tool_call("landing_ai_mcp", "create_session", {
  "user_token": token,
  "session_name": "My Product LP",
  "brand_name": "ACME",
  "product_name": "Widget Pro",
  "base_description": "Premium widget for professionals"
})

# 2. Save TA groups to session
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session.id,
  "data_json": json.dumps({"wizard_ta_groups": [selected_ta_from_generate_ta_options]})
})

# 3. Get TA group IDs
ta_statuses = mcp_tool_call("landing_ai_mcp", "get_ta_statuses", {
  "user_token": token,
  "session_id": session.id
})
# Extract ta_group_id values, e.g. ["ta_1"]

# 4. Trigger generation
mcp_tool_call("landing_ai_mcp", "generate_session", {
  "user_token": token,
  "session_id": session.id,
  "ta_group_ids_json": json.dumps(["ta_1"]),
  "requested_stripe_count": 8
})

# 5. Poll (every 10 seconds, max 30 times = 5 min)
while True:
  status = mcp_tool_call("landing_ai_mcp", "get_session", {
    "user_token": token,
    "session_id": session.id
  })
  if status == "complete": break
  if status == "error": handle_error()
  sleep(5)
```

### Pattern: Edit-and-Verify
```python
# 1. Edit
mcp_tool_call("landing_ai_mcp", "update_stripe_texts", {
  "user_token": token,
  "campaign_id": campaign_id,
  "updates_json": json.dumps([
    {"index": 0, "headline": "New Headline"}
  ])
})

# 2. Verify
stripe = mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
# Show preview to user
```

### Pattern: Multi-Platform Publish
```python
# 1. List accounts
accounts = mcp_tool_call("zereo_social_mcp", "list_accounts", {
  "user_token": token
})

# 2. Import content from LP session
mcp_tool_call("zereo_social_mcp", "import_from_session", {
  "user_token": token,
  "session_id": session_id
})

# 3. Publish to multiple platforms
# ⚠️ Each target follows SocialPublishRequest schema — key is
#    `social_account_id` (NOT `account_id`), `post_type` required
#    (ig_post / ig_story / ig_reel / fb_post / fb_story / tt_video / tt_photo)
mcp_tool_call("zereo_social_mcp", "publish_multi", {
  "user_token": token,
  "targets_json": json.dumps([
    {"social_account_id": "<ig_id>",     "post_type": "ig_post",  "caption": "...", "image_url": "https://..."},
    {"social_account_id": "<tiktok_id>", "post_type": "tt_video", "caption": "...", "video_url": "https://..."}
  ])
})
```

### Pattern: Ad Campaign Creation
```python
# 1. Get objectives (informational — you still POST campaign_objective value verbatim)
objectives = mcp_tool_call("zereo_social_mcp", "get_ad_objectives", {
  "user_token": token
})

# 2. Generate ad creative image (returns public image_url, not a creative_id).
#    GenerateAdRequest has extra="forbid". Valid fields: ta_group_id (req),
#    aspect_ratio ("9:16"/"4:5"/"1:1", default "9:16"), ad_goal
#    ("awareness"/"traffic"/"conversion", default "awareness"). Do NOT pass
#    `platform` — this endpoint just makes the image; platform is picked later
#    when you publish or create an ad campaign.
task = mcp_tool_call("landing_ai_mcp", "generate_ad", {
  "user_token": token,
  "session_id": session_id,
  "data_json": json.dumps({"ta_group_id": "ta_1", "aspect_ratio": "1:1", "ad_goal": "awareness"})
})

# 3. Poll for creative — response holds the image_url directly
result = mcp_tool_call("landing_ai_mcp", "get_ad_result", {
  "user_token": token,
  "session_id": session_id,
  "project_id": task["project_id"]
})
creative_image_url = result["image_url"]  # ← feed this to create_ad_campaign

# 4. Create campaign — follow AdCampaignCreateRequest schema EXACTLY
#    (wrong param names -> 422 "field required"; wrong objective value -> 400)
mcp_tool_call("zereo_social_mcp", "create_ad_campaign", {
  "user_token": token,
  "data_json": json.dumps({
    "social_account_id":   "<from list_accounts>",   # not account_id
    "campaign_objective":  "OUTCOME_TRAFFIC",        # not objective / TRAFFIC / CONVERSIONS
    "ad_type":             "image",                  # or "video"
    "creative_image_url":  creative_image_url,       # not creative_id
    "creative_message":    "Your ad text here",
    "cta_type":            "SHOP_NOW",
    "cta_url":             "https://landingai.info/zh-TW/lp/<campaign_id>",
    "daily_budget":        5.0,                      # USD, min $1
    "target_age_min":      25,
    "target_age_max":      40,
    "target_genders":      [0],                      # [0]=all, [1]=male, [2]=female (int, not str)
    "target_countries":    ["TW"],
    "placements":          ["facebook", "instagram"],
    "schedule_start":      "2026-04-20T00:00:00+08:00",
    "schedule_end":        "2026-05-05T00:00:00+08:00"
  })
})
```

**Valid `campaign_objective` values**: `OUTCOME_AWARENESS` / `OUTCOME_TRAFFIC` / `OUTCOME_ENGAGEMENT` / `OUTCOME_LEADS` / `OUTCOME_SALES`. Backend auto-derives the matching AdSet `optimization_goal` — do NOT pass optimization_goal yourself.

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 401 | Token expired or revoked | Ask user to re-copy a fresh AI Token from `https://salecraft.ai/{locale}/marketingx`, call `authenticate_with_token` again, retry |
| 402 | Insufficient credits | Inform user, show `get_me` balance |
| 404 | Resource not found | Verify ID exists, may have been deleted |
| 429 | Rate limited | Wait 10s, retry (max 3 retries) |
| 500 | Server error | Log error, retry once, then report to user |

## Cross-MCP Token Sharing

`landing_ai_mcp` and `zereo_social_mcp` share the same auth system. A JWT from `landing_ai_mcp > login` works for both servers.

## JSON Parameter Convention

Tools that accept complex data use `*_json` string parameters:
```python
# Correct: pass as JSON string
arguments = { "updates_json": '[ {"index": 0, "headline": "Hello"} ]' }

# Wrong: don't pass as object
arguments = { "updates_json": [{"stripe_idx": 0}] }  # This will fail
```

Always `JSON.stringify()` / `json.dumps()` before passing to `*_json` parameters.
