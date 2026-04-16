---
name: mx-reels
description: "Full Reels creation flow: consultation → script → review → generate → publish"
---

# /mx-reels — Short Video (Reels) Creation

Read `CLAUDE.md` and `prompts/WORKFLOW.md` for complete workflow reference.

Execute these phases in order:

## Phase 1: Authentication & Context

If `user_token` not available:
- Login via `mcp_tool_call("landing_ai_mcp", "login", {...})`
- Check credits via `mcp_tool_call("landing_ai_mcp", "get_me", {"user_token": token})`

If `brand_id` available from previous session, carry it forward.

## Phase 2: Consultation (FREE)

Gather product info conversationally:
- Product name & niche
- Target audience
- Key message & CTA
- Video goal (awareness / conversion / engagement)
- Preferred duration (5-60 seconds, default 10s)
- Tone & style preferences (optional)

**Show cost estimate before proceeding:**
```
影片時長: 10 秒
預估費用: 1,000 pts (100 pts/秒)
您的餘額: X pts
```

## Phase 3: Script Generation & Review

Invoke the `generate-reels` skill (Phase 1-3).
- AI generates strategy + script (~30 seconds)
- Present script as structured table
- User reviews and optionally revises
- Loop until user approves script

## Phase 4: Video Generation

Invoke the `generate-reels` skill (Phase 4-6).
- Confirm cost deduction with user
- Trigger 7-agent pipeline (2-8 minutes)
- Report progress every 30 seconds
- Present final video with links

## Phase 5: Publish (Optional)

Suggest publishing options:
- `/mx-publish` → Post to IG Reels, TikTok, Facebook
- Direct share link for manual posting

## Cross-Phase State

Carry these values across all phases — never re-ask:
- `user_token`, `brand_id`
- `session_id` (reel session)
- `script_data` (approved script)

## Quick Start Examples

**User says:** "幫我做一個10秒的產品介紹影片"
→ Start at Phase 2, gather product details

**User says:** "用上次的品牌資料做 Reels"
→ Load existing `brand_id`, skip to Phase 2

**User says:** "我有腳本了，直接製作"
→ Use `update_reel_script` to input user's script, skip to Phase 4
