# MarketingX Plugin — Known Issues

## 1. update_stripe_texts cannot clear subheadline to empty string
- **Status**: Open (backend limitation)
- **Symptom**: Calling `update_stripe_texts` with `new_text: ""` for subheadline is silently ignored
- **Workaround**: Replace subheadline with meaningful text instead of clearing it
- **Root cause**: marketing_backend may skip empty string updates for certain text fields
- **Impact**: Cannot fully remove subheadline from a stripe via MCP
- **Discovered**: 2026-04-11, during scenario test

## 2. regenerate_stripe vs regenerate_project_stripe
- **Status**: Clarified
- `regenerate_stripe(campaign_id, stripe_idx)` — sometimes doesn't trigger actual regeneration
- `regenerate_project_stripe(session_id, project_id, stripe_index, data_json)` — works, requires more params
- **Recommendation**: Use `regenerate_project_stripe` with `data_json: {"user_feedback": "..."}` for reliable regeneration
- **Discovered**: 2026-04-11

## 3. Gemini Pro Image generation can hang
- **Status**: Known backend issue
- **Symptom**: `progress_stage: "producing"` stays at 70% with `last_update` frozen
- **Workaround**: Wait up to 10 min; if still stuck, the backend may eventually complete or timeout
- **Discovered**: 2026-04-11

## 4. update_brand data_json fields not updating
- **Status**: Open (needs investigation)
- **Symptom**: `update_brand` returns success but brand fields remain unchanged
- **Workaround**: Set fields at `create_brand` time instead of updating later
- **Discovered**: 2026-04-11
