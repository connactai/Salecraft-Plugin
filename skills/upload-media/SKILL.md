---
name: upload-media
description: |
  Upload spokesperson photos, voice samples, or product images through the AI chat.
  Users can paste images directly, share a URL, or describe what they want to upload.
  Supports product photos, brand logos, spokesperson headshots, and voice samples.
  Trigger: "upload photo", "upload voice", "add spokesperson image",
  "upload product image", "add voice sample", "add my photo".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
---

# Upload Media — Spokesperson & Product Assets

You help users upload photos, voice samples, and product images to their brand profile for use in reel video generation. Assets are stored in GCS and linked to the spokesperson record.

## Prerequisites

- `user_token` from authentication (login or previous skill)
- `brand_id` — which brand to upload to
- `sp_id` — which spokesperson (for photo/voice; not needed for product images)

If any of these are missing, help the user find them:
- No token → run the **AI Token** flow (NEVER ask for email/password): hand user `https://salecraft.ai/{locale}/marketingx`, ask them to click 「複製 AI 登入 Token」 and paste `sc_live_...`, call `authenticate_with_token`
- No brand → call `list_brands` and ask user to pick
- No spokesperson → call `list_spokespersons` or `create_spokesperson`

## Upload Flow

### Phase 1: Determine What to Upload

Ask: "What would you like to upload?"
- Spokesperson photo (face/portrait for visual consistency)
- Voice sample (for ElevenLabs voice cloning — 30s+ recommended)
- Product image (for video first-frame or reference)

### Phase 2: Receive the File

**Method A — Base64 (drag-and-drop or paste image):**

The user pastes or drags a file into the chat. The AI reads it as base64.

```
# AI reads the file as base64
base64_data = <read file and encode>

# Call MCP tool
upload_spokesperson_media(
    user_token=token,
    brand_id=brand_id,
    sp_id=sp_id,
    filename="photo.jpg",
    base64_data=base64_data,
    media_type="photo",  # or "voice"
    content_type="image/jpeg"  # or "audio/wav", "audio/mpeg"
)
```

**Method B — URL (user pastes a link):**

```
# For photos:
add_spokesperson_photos(
    user_token=token,
    brand_id=brand_id,
    sp_id=sp_id,
    urls_json='["https://example.com/photo.jpg"]'
)

# For voice:
add_spokesperson_voice(
    user_token=token,
    brand_id=brand_id,
    sp_id=sp_id,
    urls_json='["https://example.com/voice.wav"]'
)
```

**Method C — Product image (no spokesperson needed):**

```
upload_base64(
    user_token=token,
    brand_id=brand_id,
    filename="product.jpg",
    base64_data=base64_data,
    asset_type="product",
    content_type="image/jpeg"
)
```

### Phase 3: Confirm Result

**Photo uploaded:**
> 代言人照片已上傳。下次製作影片時會自動使用這張照片作為角色參考。

**Voice uploaded:**
> 聲音樣本已儲存。下次製作影片時，ElevenLabs 會自動克隆這個聲音來念對白。
> 建議：錄音至少 30 秒以上，安靜環境，品質會更好。

**Product image uploaded:**
> 產品圖已上傳。製作影片時可作為開場畫面使用。

### Phase 4: Next Steps

> 接下來你可以：
> - 繼續上傳更多素材
> - `/salecraft-reels` — 開始製作影片
> - 查看品牌素材 → `get_brand_content`

## Content Type Reference

| File Type | content_type | media_type |
|-----------|-------------|------------|
| JPEG photo | image/jpeg | photo |
| PNG photo | image/png | photo |
| WebP photo | image/webp | photo |
| WAV audio | audio/wav | voice |
| MP3 audio | audio/mpeg | voice |
| M4A audio | audio/mp4 | voice |

## Limits

- Base64 via MCP: max ~2MB (LLM token window limit)
- Backend: max 10MB per file
- Voice sample: 30s-5min recommended for best clone quality
- Photos: clear face shot, well-lit, recommended

## Voice Sample Quality Tips

Tell the user:
> 聲音樣本品質建議：
> - 安靜環境錄製（不要有背景噪音）
> - 至少 30 秒，1-2 分鐘最佳
> - 正常語速說話（不要刻意放慢或加快）
> - WAV 或 MP3 格式
> - 如果檔案太大（>2MB），請壓縮或貼 URL

## Error Handling

| Error | Action |
|-------|--------|
| FILE_TOO_LARGE | Guide user to compress or use URL |
| 401 Unauthorized | Re-login |
| 404 Brand/Spokesperson not found | List brands/spokespersons, ask to pick |
| Unsupported content_type | Check file format, suggest conversion |
