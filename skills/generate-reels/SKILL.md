---
name: generate-reels
description: |
  Orchestrates short video (Reels) generation through the 7-agent AI pipeline
  (Strategist → Scriptwriter → Visual Director → Voice → Critic → Editor → Render).
  Creates a reel session, generates script, allows user review/revision, triggers
  full video generation, polls for completion, and presents the final video.
  Trigger: Phase 3 of /mx-reels, or "generate reels", "create short video",
  "make a reel", "製作短影音", "生成影片".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Reels (Short Video) Generation — AI Pipeline Orchestration

You orchestrate the 7-agent AI pipeline that generates professional short videos (Reels/Shorts/TikTok). Your job is to create the session, generate & refine the script with the user, trigger full video generation, monitor progress, and present the final output.

## Prerequisites

- `user_token` from authentication (login or previous skill)
- `brand_id` from brand-onboard (optional but recommended for brand consistency)
- Product info: name, niche, target audience, key message, CTA, goal
- Read `CLAUDE.md` for MCP tool call pattern
- Read `lib/mcp-patterns.md` for error handling

## The AI Pipeline (runs automatically in the backend)

```
Phase 0: Build Project Bible
   → Collect user inputs, brand assets, uploaded media

Phase 1: Strategist Agent (Gemini Flash)
   → Analyzes product, audience, market positioning
   → Outputs: hook framework, pacing, visual style, character roster

Phase 2: Scriptwriter Agent (Gemini Text)
   → Writes full scene-by-scene script with timing
   → Outputs: scenes (visual prompts, dialogue, overlays, timing)
   ⏸ USER REVIEWS SCRIPT HERE

Phase 3A: Visual Director (Gemini Image + Seedance/Kling)
   → Generates video for each scene/chunk
   → Outputs: video URLs per scene
Phase 3B: Voice Agent (ElevenLabs TTS)    [parallel with 3A]
   → Generates voiceover from dialogue track
   → Outputs: audio track

Phase 4: Critic Agent (Gemini Flash)
   → QC scoring (hook, pacing, clarity, visual, continuity)
   → Pass threshold: 70/100

Phase 5: Editor Agent
   → Assembles clips, captions, overlays, audio
   → Outputs: render configuration

Phase 6: Final Render (FFmpeg / Shotstack)
   → Produces final MP4 video + thumbnail

Phase 7: Post-Completion
   → Generates suggested caption + hashtags
```

## Pricing

| Duration | Cost |
|----------|------|
| 5 seconds | 500 pts |
| 10 seconds | 1,000 pts |
| 15 seconds | 1,500 pts |
| 30 seconds | 3,000 pts |
| 60 seconds | 6,000 pts |

Formula: **100 pts per second** (min 5s, max 60s)

**ALWAYS tell the user the estimated cost BEFORE triggering generation.**

---

## Phase 0: Collect Inputs

Gather these from the user conversationally (if not already known):

| Field | Required | Description |
|-------|----------|-------------|
| `product_name` | YES | Product name |
| `niche` | YES | Industry/category (e.g. skincare, food, fashion) |
| `target_audience` | YES | Who is the video for |
| `key_message` | YES | Core selling point |
| `call_to_action` | YES | What should the viewer do |
| `goal` | YES | Video goal (awareness, conversion, engagement) |
| `brand_tone` | NO | Tone of voice (warm, professional, playful) |
| `duration_seconds` | NO | Target duration (default: 10s) |
| `pacing_override` | NO | fast / normal / slow |

If `brand_id` is available from previous onboarding, mention it — the pipeline will auto-load brand assets for visual consistency.

---

## Phase 1: Create Script (Non-blocking)

This triggers Phases 0-2 (Strategy + Script) and returns immediately.

```
mcp_tool_call("landing_ai_mcp", "create_reel_script", {
  "user_token": token,
  "data_json": "{
    \"product_name\": \"...\",
    \"niche\": \"...\",
    \"target_audience\": \"...\",
    \"key_message\": \"...\",
    \"call_to_action\": \"...\",
    \"goal\": \"...\",
    \"brand_tone\": \"...\",
    \"duration_seconds\": 10
  }"
})
→ Returns: { "session_id": "reel_...", "status": "scripting" }
```

Store `session_id` — needed for all subsequent calls.

---

## Phase 2: Poll Script Status

Poll until script is ready:

```
mcp_tool_call("landing_ai_mcp", "get_reel_status", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: { "status": "SCRIPT_READY", "progress_percent": 15, ... }
```

**Polling rules:**
- Interval: every 5 seconds
- Max: 60 polls (5 minutes timeout)
- Status flow: `STRATEGIZING` → `SCRIPTING` → `SCRIPT_READY`
- Report progress to user every 15 seconds

---

## Phase 3: Present Script for Review

Once `status == "SCRIPT_READY"`, fetch and display the script:

```
mcp_tool_call("landing_ai_mcp", "get_reel_script", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: { "script_data": { "scenes": [...], "total_duration": 10, ... } }
```

**Present to user as a structured table:**

```markdown
### 腳本預覽 Script Preview

| # | 時間 | 畫面描述 | 旁白/對話 | 字幕覆蓋 |
|---|------|---------|----------|---------|
| 1 | 0-2s | [visual_description] | [spoken_text] | [overlay_text] |
| 2 | 2-5s | [visual_description] | [spoken_text] | [overlay_text] |
| ... | ... | ... | ... | ... |

總長: Xs | 場景數: N | 風格: [visual_style] | 節奏: [pacing]
```

Then ask: **「腳本看起來如何？要修改哪裡，還是直接開始製作？」**

---

## Phase 3.5: Script Revision (Optional, Iterative)

If user wants changes:

```
mcp_tool_call("landing_ai_mcp", "patch_reel_script", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"feedback\": \"User's feedback in natural language\"}"
})
```

After revision, fetch script again (Phase 3) and re-present. Loop until user approves.

For direct scene editing by user:

```
mcp_tool_call("landing_ai_mcp", "update_reel_script", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"edited_scenes\": [...]}"
})
```

---

## Phase 4: Confirm & Trigger Full Generation

Once user approves the script:

1. **Show cost estimate:**
   ```
   Duration: 10 seconds
   Estimated cost: 1,000 pts
   Current balance: X pts
   ```

2. **Get explicit confirmation:** "確認要開始製作嗎？(這將扣除 1,000 點)"

3. **Trigger generation:**
   ```
   mcp_tool_call("landing_ai_mcp", "generate_reel", {
     "user_token": token,
     "session_id": session_id,
     "data_json": "{}"
   })
   → Returns: { "status": "generating", "message": "..." }
   ```

---

## Phase 5: Poll Generation Status

Full video generation takes 2-8 minutes depending on duration and complexity.

```
mcp_tool_call("landing_ai_mcp", "get_reel_status", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: {
    "status": "GENERATING_VISUALS",
    "progress_percent": 45,
    "current_agent": "Visual Director",
    ...
  }
```

**Polling rules:**
- Interval: every 10 seconds
- Max: 60 polls (10 minutes timeout)
- Report progress to user every 30 seconds with agent name:

```
⏳ 製作進度 45% — Visual Director 正在生成影片畫面...
⏳ 製作進度 70% — Editor 正在組裝影片...
⏳ 製作進度 95% — 最終渲染中...
```

**Status flow:**
```
GENERATING_VISUALS (25-55%) → GENERATING_VOICE (parallel)
→ QC (60%) → EDITING (75%) → COMPLETED (100%)
```

---

## Phase 6: Present Final Video (MANDATORY)

Once `status == "COMPLETED"`:

```
mcp_tool_call("landing_ai_mcp", "get_reel_session", {
  "user_token": token,
  "session_id": session_id
})
→ Returns: {
    "final_video_url": "https://...",
    "thumbnail_url": "https://...",
    "suggested_caption": "...",
    "suggested_hashtags": "...",
    "duration": 10,
    "qc_report": { "overall_score": 85, ... }
  }
```

**MANDATORY output format:**

```markdown
### ✅ 影片製作完成！

🎬 **影片連結**: [點擊觀看](final_video_url)
🖼️ **縮圖**: [預覽](thumbnail_url)
⏱️ **時長**: 10 秒
📊 **品質分數**: 85/100

### 建議文案 Suggested Caption
> [suggested_caption]

### 建議 Hashtags
[suggested_hashtags]

---

**接下來你可以：**
- 📝 「修改第3個場景」 — 重新生成特定場景
- 📱 `/mx-publish` — 發佈到 IG Reels / TikTok
- 🔄 「重新製作」 — 用不同風格重做
- 💾 「存為模板」 — 儲存這個腳本格式
```

---

## Phase 7: Post-Generation Actions (Optional)

### Regenerate a Single Scene
```
mcp_tool_call("landing_ai_mcp", "regenerate_scene", {
  "user_token": token,
  "session_id": session_id,
  "scene_id": scene_id,
  "data_json": "{\"feedback\": \"Make it more dynamic\"}"
})
```

### Save as Template
```
mcp_tool_call("landing_ai_mcp", "save_as_template", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"template_name\": \"Product Showcase 10s\"}"
})
```

### Cancel In-Progress Generation
```
mcp_tool_call("landing_ai_mcp", "cancel_reel", {
  "user_token": token,
  "session_id": session_id
})
```

---

## Error Handling

| Error | Action |
|-------|--------|
| 401 Unauthorized | Re-login, retry |
| 402 Insufficient Credits | Show balance, direct to top-up page |
| 404 Session Not Found | List sessions, ask user to pick |
| 408/504 Timeout | Check status once more, report if still stuck |
| 500 Server Error | Retry once, then report error to user |
| Generation FAILED | Show error message, offer to retry or modify script |
| QC score < 70 | Inform user, suggest script modifications |

---

## SaleCraft Scope & Pricing

| Item | Cost |
|------|------|
| Script generation (Phase 1-2) | FREE preview |
| Script revision | FREE |
| Full video generation | 100 pts/second |
| Scene regeneration | ~50 pts |
| Publish to social | 5-10 pts/post |

---

## Transition Prompts (MANDATORY at decision points)

**After script ready:**
> 腳本準備好了！請看看上面的場景安排。
> - 「這樣很好，開始製作」 → 開始影片生成
> - 「第2個場景改成...」 → 修改腳本
> - 「整體風格改成更活潑」 → AI 重寫

**After video complete:**
> 影片製作完成！
> - `/mx-publish` → 發佈到社群平台
> - 「修改第X場景」 → 重新生成特定畫面
> - 「存為模板」 → 下次可以快速套用
> - `/mx-create` → 回到行銷主流程
