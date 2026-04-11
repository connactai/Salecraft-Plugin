---
name: generate-landing
description: |
  Orchestrates landing page generation through the AI agent pipeline
  (Strategist → Architect → Factory → Stripe Reflector). Creates a session,
  triggers generation, polls for completion, and verifies quality.
  Trigger: Phase 3 of /mx-create, or "generate my landing page", "create LP".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Landing Page Generation — AI Pipeline Orchestration

You orchestrate the 4-agent AI pipeline that generates professional landing pages. Your job is to set up the session correctly, trigger generation, monitor progress, and verify output quality.

## Prerequisites

- `user_token`, `brand_id`, `ta_groups`, `aspect_ratio`, `locale` from previous phases
- Read `CLAUDE.md` for tool signatures
- Read `lib/mcp-patterns.md` for the create-and-poll pattern

## The AI Pipeline (runs automatically in the backend)

```
1. Strategist Agent (Gemini Flash)
   → Analyzes brand, TA, market positioning
   → Outputs: marketing strategy, key messages, USPs

2. Architect Agent (Gemini Flash)
   → Designs page layout, stripe structure, copywriting
   → Outputs: stripe plan (headlines, body text, CTA per stripe)

3. Factory Agent (Gemini Pro Image)
   → Generates visual stripes with embedded text
   → Outputs: high-quality images with text rendered into them

4. Stripe Reflector (Gemini Flash)
   → Quality-checks each stripe
   → Flags issues: text readability, brand consistency, layout balance
```

You don't call these agents individually — `generate_session` triggers the entire pipeline.

## Phase 1: Create Session

```
mcp_tool_call("landing_ai_mcp", "create_session", {
  "user_token": token,
  "session_name": "[Product] Landing Page",
  "brand_name": brand_name,
  "product_name": product_name,
  "base_description": product_description,
  "conversation_id": conversation_id  // optional, for context continuity
})
→ Returns: { "session_id": "sess_..." }
```

## Phase 2: Configure Session

Set TA groups, aspect ratio, and locale:

```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  // Set TA config, aspect ratio, locale as needed
})
```

## Phase 2.5: Save TA Groups to Session (CRITICAL)

Before generation, TA options must be saved into the session:

### Step 1: Save selected TAs
```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_ta_groups\": [<selected TA objects from generate_ta_options>]}"
})
```

### Step 2: Get assigned TA group IDs
```
mcp_tool_call("landing_ai_mcp", "get_ta_statuses", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: [{ "ta_group_id": "ta_1", "ta_name": "...", ... }]
```

## Phase 3: Trigger Generation

```
mcp_tool_call("landing_ai_mcp", "generate_session", {
  "user_token": token,
  "session_id": session_id,
  "ta_group_ids_json": "[\"ta_1\"]",
  "requested_stripe_count": 8
})
→ Returns: { "message": "Started generation...", "status": "processing", "project_ids": ["..."] }
```

**Important**:
- `ta_group_ids_json` is REQUIRED — JSON array of IDs from Phase 2.5 Step 2.
- `requested_stripe_count`: set to 8 for 8-page LP (0 = backend default ~10).
- This is an async operation. The backend runs the 4-agent pipeline which takes 30-120 seconds.

## Phase 4: Poll for Completion

Poll every 5 seconds, max 60 attempts (5 minutes):

```
mcp_tool_call("landing_ai_mcp", "get_session", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: { "status": "generating" | "complete" | "error", ... }
```

### Progress Reporting

Keep the user informed during polling:

```
⏳ Generating your landing page...

[00:05] Strategist analyzing your brand and audience...
[00:15] Architect designing page layout and copy...
[00:30] Factory generating visual stripes...
[00:45] Quality check in progress...
[01:00] Almost done — finalizing...

✅ Generation complete! [X] stripes created.
```

### Error Handling

- **"error" status**: Report the error, offer to retry
- **Timeout (60 polls)**: "Generation is taking longer than expected. Want to wait more or check later?"
- **Partial generation**: Some stripes may fail — report which ones and offer to regenerate individually

## Phase 5: Verify Output

### List all stripes
```
mcp_tool_call("landing_ai_mcp", "list_stripes", {
  "user_token": token,
  "session_id": session_id
})
```

### Check each stripe
For key stripes (hero, CTA), inspect detail:
```
mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
```

### Quality checklist
- [ ] All expected stripes generated (typically 4-8)
- [ ] Hero stripe has clear headline and product image
- [ ] CTA stripe has clear call-to-action button
- [ ] Text is readable and correctly positioned
- [ ] Brand colors are consistent
- [ ] No error stripes or blank images

## Phase 6: Present Results

```
✅ Landing Page Generated Successfully!

Session: [session_name]
Campaign ID: [campaign_id]
Stripes: [count] generated
Aspect Ratio: [16:9 / 9:16]

Stripe Preview:
1. 🎯 Hero — "[headline text]"
2. 📋 Features — "[feature highlights]"
3. 💬 Testimonial — "[social proof]"
4. 🛒 CTA — "[call to action]"

Next steps:
A) Edit stripes → /mx-edit
B) Build homepage → /mx-homepage
C) Publish → /mx-publish
D) Generate another variant (different TA or ratio)
```

## Generating Both Aspect Ratios

If user selected "both" in Phase 2:

1. Generate 16:9 version first (steps 1-6)
2. Create a second session with same config but `aspect_ratio="9:16"`
3. Generate 9:16 version (steps 1-6)
4. Present both results together

```
✅ Both versions generated!

16:9 (Landscape): [campaign_id_landscape] — [X] stripes
9:16 (Portrait):  [campaign_id_portrait] — [Y] stripes

Which version would you like to edit first?
```

## Phase 6: Provide Preview Links (MANDATORY)

After generation completes, you MUST give the user clickable preview links:

### Step 1: Create share token
```
mcp_tool_call("landing_ai_mcp", "create_share_token", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: { "share_token": "abc123" }
```

### Step 2: Get public landing page data
```
mcp_tool_call("landing_ai_mcp", "get_public_landing_page", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: full LP config with stitched_image_url and per-stripe background_urls
```

### Step 3: Present to user
```
✅ LP 生成完畢！

📄 完整長圖預覽:
{stitched_image_url}

📑 各頁 Stripe 預覽:
1. {gra_tis_stripes[0].background_url} — {stripes[0].headline}
2. {gra_tis_stripes[1].background_url} — {stripes[1].headline}
... (list all stripes)

🔗 分享連結 Token: {share_token}
```

**This step is critical** — the user needs to SEE the LP before deciding to edit.
If multiple LPs were generated, provide links for ALL of them.

## Output

Store for subsequent phases:
- `session_id` — for content import (publishing)
- `campaign_id` — for editing and export (primary identifier)
- `share_token` — for public sharing
- `stripe_count` — number of generated stripes
- `stitched_image_url` — full LP preview image
- `stripe_urls[]` — per-stripe preview images
