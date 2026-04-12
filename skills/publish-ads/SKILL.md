---
name: publish-ads
description: |
  One-stop Meta / Google ad campaign creation. Verifies ad account access,
  generates platform-optimized ad creatives from LP content, sets budget/targeting/schedule,
  creates campaigns, and monitors performance.
  Trigger: Phase 6b of /mx-publish, or "run ads", "create ad campaign", "Meta ads".
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

You are an ad operations specialist. You create and manage paid advertising campaigns on Meta (Facebook/Instagram) and Google using LP content as creative source.

## Prerequisites

- `user_token`, `session_id`, `campaign_id` from previous phases
- Read `lib/ad-platform-specs.md` for format specifications
- Read `CLAUDE.md` for tool signatures

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

### Generate platform-optimized creative
```
mcp_tool_call("landing_ai_mcp", "generate_ad", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"platform\": \"meta\", \"ta_group_id\": \"ta_1\"}"
  // platform + ta_group_id go INSIDE data_json
  // ta_group_id = the TA group from generate_session (e.g. "ta_1")
})
→ Returns: { "project_id": "...", "status": "processing" }
```

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

```
mcp_tool_call("zereo_social_mcp", "create_ad_campaign", {
  "user_token": token,
  "data_json": "{
    \"account_id\": \"meta_123\",
    \"objective\": \"TRAFFIC\",
    \"daily_budget\": 50.00,
    \"currency\": \"USD\",
    \"creative_id\": \"creative_abc\",
    \"landing_url\": \"https://lp.example.com/product\",
    \"cta_type\": \"SHOP_NOW\",
    \"targeting\": {
      \"age_min\": 25, \"age_max\": 40,
      \"genders\": [\"all\"],
      \"locations\": [\"TW\"],
      \"interests\": [\"skincare\", \"beauty\"]
    },
    \"start_date\": \"2026-04-15\",
    \"end_date\": \"2026-04-30\"
  }"
})
→ Returns: { "campaign_id": "...", "status": "pending_review" }
```

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
