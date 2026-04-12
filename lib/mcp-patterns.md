# MCP Call Patterns Reference

## Universal Call Pattern

All MCP tools are called through the Service System Deep Research proxy:

```
mcp__claude_ai_Service_System_Deep_Research__mcp_tool_call(
  server_name = "<mcp_server_name>",
  tool_name   = "<tool_name>",
  arguments   = { "param1": "value1", "param2": "value2" }
)
```

## Authentication Flow

### Step 1: Login
```
mcp_tool_call(
  server_name = "landing_ai_mcp",
  tool_name   = "login",
  arguments   = { "email": "<user_email>", "password": "<user_password>" }
)
→ Returns: { "access_token": "eyJ...", "token_type": "bearer" }
```

### Step 2: Use Token
Every subsequent call includes `user_token`:
```
mcp_tool_call(
  server_name = "landing_ai_mcp",
  tool_name   = "list_brands",
  arguments   = { "user_token": "eyJ..." }
)
```

### Step 3: Re-login (on 401)
```
# NOTE: login does NOT return a refresh_token. When you get 401, simply re-login:
mcp_tool_call(
  server_name = "landing_ai_mcp",
  tool_name   = "login",
  arguments   = { "email": "...", "password": "..." }
)
→ Returns: new { "access_token": "eyJ...", "token_type": "bearer" }
# Token expires in ~12 hours.
```

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
mcp_tool_call("zereo_social_mcp", "publish_multi", {
  "user_token": token,
  "targets_json": json.dumps([
    {"account_id": "meta_123", "caption": "Check out our new product!"},
    {"account_id": "tiktok_456", "caption": "New drop! Link in bio"}
  ])
})
```

### Pattern: Ad Campaign Creation
```python
# 1. Get objectives
objectives = mcp_tool_call("zereo_social_mcp", "get_ad_objectives", {
  "user_token": token
})

# 2. Generate ad creative
task = mcp_tool_call("landing_ai_mcp", "generate_ad", {
  "user_token": token,
  "session_id": session_id,
  "platform": "meta"  # or "google"
})

# 3. Poll for creative
result = mcp_tool_call("landing_ai_mcp", "get_ad_result", {
  "user_token": token,
  "task_id": task.id
})

# 4. Create campaign
mcp_tool_call("zereo_social_mcp", "create_ad_campaign", {
  "user_token": token,
  "data_json": json.dumps({
    "account_id": "meta_123",
    "objective": "CONVERSIONS",
    "daily_budget": 50.00,
    "creative_id": result.creative_id,
    "landing_url": "https://...",
    "start_date": "2026-04-15",
    "end_date": "2026-04-30"
  })
})
```

## Error Handling

| Error | Cause | Resolution |
|-------|-------|------------|
| 401 | Token expired | Call `refresh_tokens`, retry |
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
