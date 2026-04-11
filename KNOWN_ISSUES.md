# MarketingX Plugin — Known Issues

## RESOLVED

### 1. update_stripe_texts was silently ignoring updates
- **Root cause**: Plugin docs used wrong JSON format: `{"stripe_idx": N, "text_key": "...", "new_text": "..."}`
- **Fix**: Backend expects `{"index": N, "headline": "...", "subheadline": "..."}` — fields set directly
- **Status**: FIXED in plugin skills + CLAUDE.md

### 2. update_brand not persisting extra fields (tagline, value_proposition, etc.)
- **Root cause**: MCP wrapper `update_brand` only accepted 5 fixed params, ignored `data_json`
- **Fix**: Added `data_json` parameter support to `update_brand` in Service_system MCP wrapper
- **Status**: FIXED in Service_system/landing_ai_mcp/tools/brands.py

### 3. regenerate_stripe 3-step flow was not documented
- **Root cause**: Plugin skill only called `regenerate_stripe` but missed the required `complete_regeneration` step
- **Fix**: Documented full 3-step flow (trigger → poll → confirm) in edit-landing skill + CLAUDE.md
- **Status**: FIXED in plugin docs

## OPEN

### 4. Gemini renders typography metadata as visible text in images
- **Status**: Open (backend prompt engineering issue)
- **Symptom**: Font/size/color debug info (e.g., `Noto Sans TC, 10pt, 20, 600 rgba(255,255,255,0.9)`) rendered as visible text in stripe images
- **Root cause**: Factory agent's image prompt leaks stripe_typography config values into the Gemini image generation prompt, and Gemini renders them as actual text
- **Impact**: Unprofessional-looking stripes with debug-style text
- **Fix needed**: In `marketing_backend/agents/factory.py` or `core/prompts.py` — strip typography metadata from the text content passed to Gemini
- **Workaround**: Regenerate the stripe — sometimes Gemini doesn't include it

### 5. Gemini Pro Image generation can hang
- **Status**: Known backend issue (not plugin-related)
- **Symptom**: `progress_stage: "producing"` stays at 70% with `last_update` frozen
- **Workaround**: Wait up to 10 min; if still stuck, the backend may eventually complete or timeout
