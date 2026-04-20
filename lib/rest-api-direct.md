# SaleCraft REST API — Direct Call Reference (No MCP Required)

**For LLMs without MCP tools.** If your runtime doesn't expose `landing_ai_mcp` / `zereo_social_mcp` as callable tools, you can still execute every paid action by calling the SaleCraft backend over plain HTTPS. This document is your contract.

> **For users:** there is still nothing to install. The user only:
> 1. Opens `https://salecraft.ai/{locale}/marketingx`
> 2. Clicks 「複製 AI 登入 Token」
> 3. Pastes `sc_live_…` back to chat
>
> You (the AI) do everything else with `fetch` / `requests` / your platform's HTTP tool.

---

## Base URL

```
https://marketing-backend-v2-s6ykq3ylca-de.a.run.app
```

This is the production REST endpoint — it works right now. Use it directly in your HTTP calls.

**A friendlier `https://api.salecraft.ai` is planned but not yet active** (the `salecraft.ai` domain hasn't been verified in this GCP project yet — see the operator note at the bottom of this file). Once it's set up, this doc will be updated; until then, `*.run.app` is correct.

**URL discipline for what you SAY to the user:** the Cloud Run URL is fine to use silently in your HTTP requests, but if you ever need to mention an URL to the user (top-up, login, account settings), only use `https://salecraft.ai/{locale}/marketingx`. Do not display the `*.run.app` URL in chat.

## Auth (4 steps)

### 1. User generates an AI Token

Tell the user (replace `{locale}` with their language code: `zh-TW`, `en`, `ja`, `ko`, `vi`, `fr`, `th`, `es`, `pt`, `ar`):

> 「① 開這個連結登入：https://salecraft.ai/{locale}/marketingx
> ② 點頁面上的「複製 AI 登入 Token」按鈕
> ③ 把 `sc_live_…` 貼回來給我」

### 2. Exchange `sc_live_*` for an `access_token`

```http
POST https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/auth/ai-token/exchange
Content-Type: application/json

{ "ai_token": "sc_live_eyJhbGc..." }
```

**Response 200:**
```json
{
  "access_token": "eyJhbGc...",
  "scope": "ai_agent"
}
```

**Response 401** (`{"detail": {"code": "INVALID_AI_TOKEN", ...}}`) → ask user to re-copy a fresh token from marketingx. **Never** fall back to email/password.

### 3. Use the access token in every subsequent call

```http
Authorization: Bearer eyJhbGc...
```

The token is valid ~12 hours. On 401 with `INVALID_AI_TOKEN`, repeat steps 1-2.

### 4. Scope limits

`access_token` from this flow has `scope=ai_agent` and **cannot** call: `DELETE /auth/account`, password change, large topup. These return `403 SCOPE_FORBIDDEN` — direct the user to do those on the marketingx page itself.

---

## Endpoint Catalog

All endpoints below assume `Authorization: Bearer <access_token>` unless marked **public**.

### Account & profile
| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/auth/me` | User profile + remaining credits |
| `PUT` | `/auth/me` | Update profile |
| `GET` | `/auth/me/settings` | User settings |
| `PUT` | `/auth/me/settings` | Update settings |
| `POST` | `/auth/logout` | Invalidate access_token |
| `POST` | `/auth/ai-token/exchange` | **Public.** Exchange `sc_live_*` for access_token |

### Brands (FREE consultation + asset management)
| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/brands/` | List user's brands |
| `POST` | `/brands/` | Create brand |
| `GET` | `/brands/{brand_id}` | Brand detail |
| `PUT` | `/brands/{brand_id}` | Update brand |
| `POST` | `/brands/analyze-url` | Auto-extract brand from URL (FREE) |
| `POST` | `/brands/{brand_id}/gap-analysis` | Identify missing assets (FREE) |
| `POST` | `/brands/{brand_id}/assets/upload-base64` | Upload image as base64 |
| `POST` | `/brands/{brand_id}/assets/upload-url` | Get signed URL for direct upload |
| `GET` | `/brands/{brand_id}/assets` | List brand assets |
| `POST` | `/brands/{brand_id}/knowledge/from-url` | Add knowledge from URL |
| `POST` | `/brands/{brand_id}/knowledge/text` | Add knowledge from text |

### Sessions (Landing Page generation)
| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/sessions/` | Create session (draft, no charge) |
| `GET` | `/sessions/` | List user sessions |
| `GET` | `/sessions/{session_id}` | Session detail (poll status here) |
| `PUT` | `/sessions/{session_id}` | Update wizard data (TA groups, brand info) |
| `GET` | `/sessions/{session_id}/ta-statuses` | TA group statuses |
| `POST` | `/sessions/{session_id}/generate` | **PAID.** Trigger LP generation pipeline |
| `DELETE` | `/sessions/{session_id}` | Delete session |
| `GET` | `/sessions/{session_id}/content` | Get all stripes content for the session |

### Quick Ads & Carousel (PAID)
| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/sessions/{session_id}/generate-ad` | Single ad image (~5 min, 200 pts) |
| `GET` | `/sessions/{session_id}/ad-result/{project_id}` | Poll ad result |
| `POST` | `/sessions/{session_id}/generate-carousel` | Multi-image carousel (~8 min, 300+100×N pts) |
| `GET` | `/sessions/{session_id}/carousel-result/{project_id}` | Poll carousel result |

### Landing Page editing (PAID for stripe regeneration)
| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/landing/{campaign_id}` | Full LP detail |
| `GET` | `/landing/public/{campaign_id}` | Public LP view (share link) |
| `POST` | `/landing/{campaign_id}/regenerate-config` | Regenerate config |

(Stripe-level editing endpoints — text/image/overlay/crop/regenerate — are exposed under `/landing/...` and `/sessions/{id}/projects/{pid}/stripes/...`. See `skills/edit-landing/SKILL.md` for the per-action mappings.)

### Reels (short video, PAID at 100 pts/sec)
| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/reels/` | Create reel session |
| `POST` | `/reels/{id}/generate` | Trigger reel generation |
| `GET` | `/reels/{id}` | Poll reel status |

### Spokespersons (for Reels)
| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/spokespersons` | List spokespersons |
| `POST` | `/spokespersons` | Create spokesperson |
| `POST` | `/spokespersons/{sp_id}/upload-photo` | Upload face photo |
| `POST` | `/spokespersons/{sp_id}/upload-voice` | Upload voice sample |

### Pricing & credits
| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/pricing/plans` | List plans |
| `GET` | `/pricing/balance` | Current credit balance |

### Social publishing (zereo_social — separate base if mounted under `/api/social`)

If the social MCP server's REST counterpart is hosted under the same backend, use those paths. If it's a separate service, the user should be told to publish via the marketingx dashboard directly. **Do not guess endpoint paths** — call `GET /sessions/{id}/content` to fetch the assets, then either:
- Hand the user the asset URL + caption and let them post via marketingx, OR
- If `POST /api/social/publish` is documented in `/docs`, use it

---

## Standard request/response patterns

### Create session → poll until done

```python
# 1. Exchange AI Token (once per session)
r = httpx.post("https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/auth/ai-token/exchange",
               json={"ai_token": "sc_live_..."})
access_token = r.json()["access_token"]
H = {"Authorization": f"Bearer {access_token}"}

# 2. Create session
r = httpx.post("https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/", headers=H, json={
    "session_name": "My Product LP",
    "brand_name": "ACME",
    "product_name": "Widget Pro",
    "base_description": "Premium widget for professionals",
})
session_id = r.json()["id"]

# 3. Update wizard data with selected TAs
httpx.put(f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}", headers=H, json={
    "wizard_ta_groups": [...],   # from /generate-ta-options if used
    "wizard_shared_data": {...},
})

# 4. Trigger generation
httpx.post(f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}/generate", headers=H, json={
    "ta_group_ids": ["ta_1"],
    "requested_stripe_count": 8,
})

# 5. Poll every 10s, max 60 attempts (= 10 min)
for _ in range(60):
    time.sleep(10)
    r = httpx.get(f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}", headers=H)
    status = r.json().get("status")
    if status == "complete":
        break
    if status == "error":
        raise RuntimeError(r.json().get("error_message"))
```

### Quick ad image (~5 min, 200 pts)

```python
# 1. Trigger
r = httpx.post(f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}/generate-ad",
               headers=H, json={"platform": "meta", "ta_group_id": "ta_1"})
project_id = r.json()["project_id"]

# 2. Poll every 30s, max 20 attempts (= 10 min)
for _ in range(20):
    time.sleep(30)
    r = httpx.get(
        f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}/ad-result/{project_id}",
        headers=H,
    )
    if r.json().get("status") == "completed":
        image_url = r.json()["image_url"]
        break
```

### Carousel (multi-image, ~8 min)

```python
# ⚠️ The field is `num_images` — NOT `stripe_count`, `count`, `slide_count` etc.
# Backend's GenerateCarouselRequest does not enforce extra="forbid", so wrong
# field names are silently ignored and the default num_images=5 is used.
# Always pass the exact field name `num_images`.
r = httpx.post(f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}/generate-carousel",
               headers=H, json={
    "ta_group_id": "ta_1",
    "num_images": 5,                      # 1-10, default 5
    "aspect_ratio": "1:1",                # one of: "9:16", "4:5", "1:1"
    "ad_goal": "awareness",               # one of: "awareness", "traffic", "conversion"
    "carousel_narrative": "hook → features → proof → CTA",
})
project_id = r.json()["project_id"]

for _ in range(20):
    time.sleep(30)
    r = httpx.get(
        f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/sessions/{session_id}/carousel-result/{project_id}",
        headers=H,
    )
    body = r.json()
    if body["status"] == "completed":
        image_urls = body["image_urls"]   # list[str]
        ad_copy = body["ad_copy"]          # {headline, body_text, hashtags, cta_text}
        break
```

### Upload an image (base64)

```python
r = httpx.post(
    f"https://marketing-backend-v2-s6ykq3ylca-de.a.run.app/brands/{brand_id}/assets/upload-base64",
    headers=H,
    json={
        "filename": "product.jpg",
        "base64_data": "<base64_string>",
        "asset_type": "product",          # whitelist: product/logo/spokesperson/certification
        "content_type": "image/jpeg",     # whitelist: jpeg/png/webp/gif/pdf
    },
)
public_url = r.json()["public_url"]
```

---

## Error handling

| Status | Meaning | What to do |
|--------|---------|-----------|
| `401` w/ `INVALID_AI_TOKEN` | AI Token invalid or expired | Ask user to re-copy from marketingx; **never** ask for password |
| `401` (other) | access_token expired | Re-exchange the AI Token |
| `402 PAYMENT_REQUIRED` | Out of credits | Tell user balance + topup link `https://salecraft.ai/{locale}/marketingx` |
| `403 SCOPE_FORBIDDEN` | Sensitive op blocked for AI Token scope | Tell user to do this on marketingx directly |
| `404` | Resource missing | Verify ID; resource may have been deleted |
| `422` | Pydantic validation failure | Read `detail`, fix payload — schema is strict, no unknown fields |
| `429` | Rate limited | Backoff: 10s, 30s, 60s; respect `Retry-After` header |
| `5xx` | Server error | Retry once with backoff, then report to user |

---

## Rate limits

- Default: **120 req/min** per access_token
- Auth endpoints (`/auth/token`, `/auth/register`, `/auth/ai-token/exchange`): **15/min** per email
- Public endpoints: **60/min** per IP

If you hit 429, you're calling too aggressively — increase polling intervals (30s → 60s for long ops).

---

## CORS

`Authorization`, `Content-Type`, `Accept`, `Origin`, `X-Requested-With`, `X-Client-Timezone` are all allowed. Server-to-server calls from LLM infra (no Origin) work fine.

---

## OpenAPI / Swagger

`/openapi.json` and `/docs` are **disabled in production** for security. The endpoint catalog above is your reference. If you need an endpoint not listed (e.g. a stripe-edit endpoint), check the [SaleCraft Plugin source](https://github.com/connactai/Salecraft-Plugin/tree/master/skills) — every skill's SKILL.md documents the MCP tool names, which map 1:1 to REST resource paths.

---

## Forbidden — do NOT call

These endpoints are deprecated for AI use; the AI must never handle credentials:

```
POST /auth/token                    (login with email+password)
POST /auth/register                 (create account with email+password)
POST /auth/forgot-password
POST /auth/reset-password
GET  /auth/verify-email
POST /auth/resend-verification
DELETE /auth/account                (returns 403 SCOPE_FORBIDDEN for AI Token anyway)
```

Account creation, password reset, and email verification all happen on `https://salecraft.ai/{locale}/marketingx` — the user does it, then comes back with an AI Token.

---

## Setup note for the operator (Landing AI)

To upgrade the API base URL from `https://marketing-backend-v2-s6ykq3ylca-de.a.run.app` (current) to `https://api.salecraft.ai` (target), do these steps in order:

**Step 1 — Verify `salecraft.ai` ownership in Google Search Console** (required because GCP refuses to map a domain you don't own):
- Visit https://search.google.com/search-console
- Add property: `salecraft.ai`
- Choose "Domain" verification → copy the TXT record GCP gives you
- Add that TXT record at your DNS provider for `salecraft.ai`
- Wait for verification (usually < 5 min)

**Step 2 — Create the Cloud Run domain mapping**:
```
gcloud beta run domain-mappings create \
  --service=marketing-backend-v2 \
  --domain=api.salecraft.ai \
  --region=asia-east1 \
  --project=gen-lang-client-0747655769
```

**Step 3 — Add the CNAME at your DNS provider** (Cloud Run will print the exact target after step 2 — usually `ghs.googlehosted.com` for subdomains).

**Step 4 — Update this doc**: replace every `https://marketing-backend-v2-s6ykq3ylca-de.a.run.app` with `https://api.salecraft.ai` (single grep+sed in this file). Commit, push.

Until step 4, the `*.run.app` URL is the correct primary base. Both URLs serve the same backend, so a user mid-session won't be disrupted by the swap.
