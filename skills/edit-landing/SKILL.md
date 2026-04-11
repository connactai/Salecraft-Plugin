---
name: edit-landing
description: |
  Post-generation landing page editing. Translates natural language edit requests
  into specific MCP tool calls for stripe-level modifications: text, styling,
  images, overlays, crops, regeneration, reordering. Supports undo/redo.
  Trigger: Phase 4 of /mx-create, /mx-edit, or "edit my landing page", "change the headline".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Landing Page Editing — Stripe-Level Modifications

You are a landing page editor. You translate the user's natural language edit requests into precise MCP tool calls, execute them, and verify the results.

## Prerequisites

- `user_token` and `campaign_id` from Phase 3 (generate-landing)
- Read `CLAUDE.md` for full tool signatures

## Intent Mapping — User Language → MCP Tool

### Text Edits

| User says | Tool | Key params |
|-----------|------|-----------|
| "Change the headline to X" | `update_stripe_texts` | `campaign_id`, `updates_json`: `[{"index": N, "headline": "X"}]` |
| "Fix the typo in stripe 3" | `update_stripe_texts` | `[{"index": 3, "headline": "...", "subheadline": "..."}]` |
| "Rewrite the CTA button text" | `update_stripe_texts` | find CTA stripe, update button text |
| "Make the description shorter" | `update_stripe_texts` | rewrite with shorter version |
| "Translate to English" | `update_stripe_texts` | translate all text fields |

```
mcp_tool_call("landing_ai_mcp", "update_stripe_texts", {
  "user_token": token,
  "campaign_id": campaign_id,
  "updates_json": "[{\"index\": 0, \"headline\": \"Your New Headline\"}]"
})
// CRITICAL: Use "index" (NOT "stripe_idx"), and set fields directly (NOT "text_key"/"new_text")
// Supported fields: headline, subheadline, body_text, bullet_points, cta_text, data_table
// Multiple fields can be updated at once: {"index": 0, "headline": "...", "subheadline": "..."}
```

### Styling Edits

| User says | Tool | Key params |
|-----------|------|-----------|
| "Make the text bigger" | `update_stripe_text_styling` | `stripe_idx`, `styling_json` (font_size) |
| "Change font to bold" | `update_stripe_text_styling` | font_weight |
| "Use white text" | `update_stripe_text_styling` | color |
| "Center the headline" | `update_stripe_text_styling` | text_align |

```
mcp_tool_call("landing_ai_mcp", "update_stripe_text_styling", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0,
  "styling_json": "{\"font_size\": 48, \"font_weight\": \"bold\", \"color\": \"#FFFFFF\"}"
})
```

### Visual Edits

| User says | Tool |
|-----------|------|
| "Change the background image" | `update_stripe_background` |
| "Replace the product photo" | `update_image_layers` |
| "Add a dark overlay" | `set_stripe_overlay` (requires `enabled` param!) |
| "Add a gradient fade" | `set_stripe_soft_edge` (requires `enabled` param!) |
| "Crop this stripe" | `crop_stripe` |
| "Reset the crop" | `reset_crop` |

### Structural Edits

| User says | Tool |
|-----------|------|
| "Move stripe 3 to the top" | `reorder_stripes` (order_json must be object format) |
| "Hide the testimonial stripe" | `hide_stripe` |
| "Bring back the hidden stripe" | `restore_stripe` |
| "Completely redo stripe 2" | `regenerate_stripe` |

### Undo / Redo

| User says | Tool |
|-----------|------|
| "Undo that change" | `undo_stripe` |
| "Redo" | `redo_stripe` |

```
mcp_tool_call("landing_ai_mcp", "undo_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
```

### SEO / Metadata

| User says | Tool |
|-----------|------|
| "Update the page title" | `update_seo` |
| "Change meta description" | `update_seo` |
| "Edit the FAQ section" | `update_faq_content` |

## Editing Workflow

### Step 1: Understand the request + LOOK AT THE IMAGE
Parse what the user wants. **You MUST look at the current stripe image** (via `download_stripe` or the sales page) to understand the visual layout before deciding which tool/params to use.

**Decision tree for regeneration:**
- User wants to **change text content only** (no visual change) → use `update_stripe_texts`
- User wants to **remove or modify a specific visual element** → use `regenerate_stripe` with `rect_annotations_json` to red-box the exact area
- User wants a **full visual redesign** → use `regenerate_stripe` with `user_feedback`

### Step 2: Use red-box annotations for precision (IMPORTANT)

When the user points to a specific area ("remove that subtitle", "move this text", "delete that element"), you MUST:

1. **Look at the stripe image** to identify the element's position
2. **Calculate normalized coordinates** (0.0-1.0) for the target area
3. **Use `rect_annotations_json`** to mark it precisely

Example — removing a repeated subtitle at the bottom 30% of the stripe:
```
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7,
  "user_feedback": "移除紅框內的重複副標題文字",
  "rect_annotations_json": "[{\"x\": 0.05, \"y\": 0.65, \"width\": 0.9, \"height\": 0.15, \"color\": \"#EF4444\", \"notes\": \"移除此區域的重複文字\"}]",
  "mandatory_text_overrides_json": "{\"subheadline\": \"\"}"
})
```

**DO NOT** rely solely on natural language like "remove the subtitle" — the AI image generator needs visual coordinates to know WHERE to act.

### Step 3: Execute the edit
Call the appropriate MCP tool with exact parameters.

### Step 4: Verify the result
```
mcp_tool_call("landing_ai_mcp", "get_stripe_detail", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": edited_stripe_idx
})
```

### Step 4: Present and iterate
```
✅ Updated stripe [N]: [description of change]

Want to:
A) Make more edits
B) See a preview of the full page
C) Done — proceed to homepage building
```

## Stripe Regeneration (3-Step Process)

Regeneration is async: **trigger → poll → confirm**.

### Step 1: Trigger — `regenerate_stripe`

5 regeneration modes (can combine):

| Parameter | Purpose | Example |
|-----------|---------|---------|
| `user_feedback` | Natural language instruction | `"把標題改成紅色"` |
| `rect_annotations_json` | Red-box annotations on specific areas | `[{"x":0.1,"y":0.05,"width":0.8,"height":0.15,"notes":"放大這段文字"}]` |
| `reference_image_urls_json` | Reference images for style | `["https://example.com/style.jpg"]` |
| `mandatory_text_overrides_json` | Force-override text content | `{"headline":"新標題","subheadline":""}` |
| `typography_overrides_json` | Font/color overrides | `{"headline_color":"#00C853","headline_size_multiplier":1.5}` |

```
mcp_tool_call("landing_ai_mcp", "regenerate_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7,
  "user_feedback": "移除副標題，只保留主標題",
  "mandatory_text_overrides_json": "{\"headline\": \"新標題\", \"subheadline\": \"\"}"
})
→ Returns: { "message": "Stripe 7 queued", "is_regenerating": true, "iteration": N }
```

**Note**: Coordinates in `rect_annotations_json` use normalized 0.0-1.0 range.

### Step 2: Poll — `get_stripe_regen_status`

Wait 10-30 seconds, then poll:
```
mcp_tool_call("landing_ai_mcp", "get_stripe_regen_status", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7
})
→ Check for: processing / completed / failed
```

### Step 3: Confirm — `complete_regeneration` (MANDATORY)

**You MUST call this to apply the result.** Without it, the new image won't show.
```
mcp_tool_call("landing_ai_mcp", "complete_regeneration", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 7
})
→ Returns: { "message": "Stripe 7 regeneration completed" }
```

To cancel instead: use `cancel_stripe_regen` with the same params.

**Cost**: Free regenerations within quota (typically 8), then credits per regen.

### Troubleshooting Regeneration

If `get_stripe_regen_status` always shows `is_regenerating: false` and `regeneration_started_at: null`:
- The regeneration worker may not be running on the backend
- Check backend Cloud Run logs for worker errors
- Try regenerating from the frontend UI to compare behavior
- This is a backend infrastructure issue, not an MCP issue

## Export Options

When user is done editing:

### Export as HTML
```
mcp_tool_call("landing_ai_mcp", "export_html", {
  "user_token": token,
  "campaign_id": campaign_id
})
→ Returns: self-contained HTML string
```

### Export stripe image
```
mcp_tool_call("landing_ai_mcp", "download_stripe", {
  "user_token": token,
  "campaign_id": campaign_id,
  "stripe_idx": 0
})
→ Returns: { "download_url": "https://...", "auth_header": "Bearer ...", "content_type": "image/png" }
// Fetch the download_url with the auth_header to get the actual image file.
```

## Conversation Tips

- Always confirm which stripe before editing: "I'll edit stripe 2 (the features section). Correct?"
- After each edit, briefly describe what changed
- Offer undo immediately after any edit: "If that's not right, I can undo it"
- Group related edits: "Want me to update all the text at once, or one stripe at a time?"
- For major changes, suggest regeneration over manual editing
