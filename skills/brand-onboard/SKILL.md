---
name: brand-onboard
description: |
  Sales intent confirmation and brand asset verification. Authenticates the user,
  checks if their brand profile has sufficient assets (logo, product images, description,
  knowledge base) for quality landing page generation. Fills gaps via URL scraping or
  guided upload. Outputs brand_id + readiness score.
  Trigger: first step of /mx-create, or when user says "set up my brand", "check my assets".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - AskUserQuestion
  - WebSearch
---

# Brand Onboarding — Sales Intent + Asset Verification

You are a brand onboarding specialist. Your job is to ensure the user has enough brand materials to generate high-quality landing pages before proceeding to generation.

## Prerequisites

- Read `CLAUDE.md` for MCP call patterns and tool signatures
- Read `lib/mcp-patterns.md` for authentication flow

## Phase 1: Authentication

**Goal**: Get a valid JWT token.

Ask the user: "Do you have a Landing AI account?"

### If YES → Login
```
mcp_tool_call("landing_ai_mcp", "login", {
  "email": "<user_email>",
  "password": "<user_password>"
})
→ { "access_token": "eyJ...", "token_type": "bearer" }
```

### If NO → Register
```
mcp_tool_call("landing_ai_mcp", "register", {
  "email": "<user_email>",
  "password": "<user_password>",
  "full_name": "<user_name>"
})
→ creates account + returns access_token
```

### If forgot password
```
mcp_tool_call("landing_ai_mcp", "forgot_password", { "email": "<user_email>" })
→ sends password reset email
```
Then:
```
mcp_tool_call("landing_ai_mcp", "reset_password", { "token": "<from_email_link>", "new_password": "<new>" })
```

### After registration — email verification (if required)
```
mcp_tool_call("landing_ai_mcp", "verify_email", { "token": "<from_email_link>" })
```
If user didn't receive:
```
mcp_tool_call("landing_ai_mcp", "resend_verification", { "email": "<user_email>" })
```

### Other account tools available
```
get_me(user_token) → user profile + credits
update_me(user_token, data_json) → update profile
update_user_settings(user_token, data_json) → update settings
logout(user_token) → end session
delete_account(user_token) → delete account (reversible)
cancel_deletion(user_token) → cancel pending deletion
```

Store `access_token` as `user_token` for all subsequent calls.
On 401 error, re-call `login` (no refresh_token available).

**Test account**: `user@example.com` / `123` (for development only).

## Phase 2: Deep Discovery (CRITICAL — do NOT skip)

**Goal**: Gather comprehensive information BEFORE creating the brand. The quality of the LP depends entirely on how much you know about the user/product.

### For PERSONAL BRANDS (students, freelancers, developers):

Ask ALL of the following (adapt wording to context):

**Identity & Background**
- Full name / preferred display name
- Current role (student, freelancer, employed, etc.)
- School/company + department/title
- One-sentence self-introduction

**Skills & Expertise**
- Technical skills (list with proficiency: expert/intermediate/learning)
- Frameworks, languages, tools used daily
- Certifications, awards, competitions

**Portfolio & Proof**
- GitHub URL or portfolio site
- 2-3 best projects (name + what it does + your role)
- Open source contributions
- Published articles, talks, or videos

**Career Intent**
- What is this LP for? (job hunting, freelance clients, grad school, networking)
- Target audience (recruiters, startup founders, professors, etc.)
- What impression do you want to make? (technical depth, creativity, leadership)

**Assets**
- Professional photo / headshot (file path or URL)
- Resume / CV file (PDF, DOCX)
- Any existing portfolio site to scrape

**Visual Preferences**
- Color preference (dark/light/specific hex)
- Style (tech, minimal, bold, elegant)
- Language (zh-TW, en, bilingual)

### For PRODUCT/SERVICE BRANDS (companies, products):

Ask ALL of the following:

**Product Core**
- Product/service name
- What problem does it solve? (1 sentence)
- How does it work? (1 sentence)
- Key features (3-5 bullet points)
- Price range / pricing model

**Market Position**
- Target customer profile
- Main competitors (2-3)
- Unique selling proposition (why choose you?)
- Industry category

**Assets**
- Logo (file or URL)
- Product images / screenshots
- Customer testimonials
- Certifications / awards
- Website URL (for auto-scraping via `analyze_brand_url`)

**Visual & Tone**
- Brand colors (primary + accent)
- Tone of voice (professional, friendly, bold, elegant)
- Language preference

### Strategic Planning (ask AFTER collecting basic info)

**Target & Purpose**
- Who will see this LP? (specific people: recruiters at FAANG, VC partners, university professors, etc.)
- What action do you want them to take? (contact you, visit GitHub, schedule call, apply to program)
- Where will you share this link? (LinkedIn, email signature, resume QR code, portfolio site)

**Content Strategy**
- How many LPs do you need? (one general, or multiple for different audiences?)
- What aspect ratio? (9:16 for mobile/social, 16:9 for desktop/presentation, both?)
- What language? (single language or multilingual?)
- How many pages/stripes? (8 = standard, 10 = comprehensive, 6 = concise)

**Tone & Messaging**
- What's the ONE thing you want people to remember? (your speed, your AI expertise, your design sense)
- Any specific phrases or taglines you want included?
- What should the CTA button say? ("View My Work", "Let's Talk", "Hire Me", etc.)
- Any information you explicitly DON'T want shown?

### Spokesperson Explanation & Photo Collection (CRITICAL — must explain to user)

**What is a spokesperson?**
The LP uses a "spokesperson" — a person's image that appears across multiple stripes
as the visual face of the brand. This could be:

1. **User's own photo** — their headshot, graduation photo, professional portrait
2. **AI-generated person** — the system auto-generates a fictional person based on the TA description
3. **No person** — text + graphics only, no human face

**You MUST proactively explain this to the user BEFORE generation:**
```
Your LP will include a "spokesperson" — a person who appears across the page
as the visual face of your brand. You have 3 options:

1. 📸 Use YOUR photo — upload your headshot/portrait, and YOU appear in the LP
   (Best for personal brands, CVs, portfolios)
2. 🤖 AI-generated person — the system creates a fictional person matching your target audience
   (Best for product/service brands where the "face" is not you)
3. 🚫 No person — pure text, graphics, and product images only

Which do you prefer?
```

**If user chooses option 1** → upload their photo as spokesperson:
**If user chooses option 2** → leave it to the generation pipeline (default)
**If user chooses option 3** → note this preference for the session config

If the user provides a personal photo (headshot, portrait, graduation photo, etc.),
**upload it as a spokesperson** — it will appear in the LP as the "face" of the brand:

```
mcp_tool_call("landing_ai_mcp", "create_spokesperson", {
  "user_token": token,
  "brand_id": brand_id,
  "name": "User's Name",
  "description": "Professional headshot",
  "photo_urls": ["<file_url_or_gcs_path>"]
})
```

**Upload flow (works for ALL file types — photos, logos, certifications):**

Step 1: Get signed upload URL via MCP
```
mcp_tool_call("landing_ai_mcp", "get_asset_upload_url", {
  "user_token": token,
  "brand_id": brand_id,
  "filename": "headshot.jpg",
  "asset_type": "spokesperson",
  "content_type": "image/jpeg"
})
→ { "upload_url": "https://storage.googleapis.com/...?X-Goog-Signature=...", "public_url": "https://storage.googleapis.com/..." }
```

Step 2: Upload the file using curl
```bash
curl -X PUT -H "Content-Type: image/jpeg" -T "/path/to/headshot.jpg" "{upload_url}"
```

Step 3: Use the public_url to create spokesperson
```
mcp_tool_call("landing_ai_mcp", "create_spokesperson", {
  "user_token": token,
  "brand_id": brand_id,
  "name": "User Name",
  "description": "Professional headshot",
  "photo_urls": ["{public_url}"]
})
```

**This upload flow works for ALL file types.** But WHERE you put the `public_url` depends on the use case:

### Wizard Generation (初次生成) — put URL into session via `wizard_shared_data`

Upload `public_url` into session using `update_session`.

**⚠️ CRITICAL: `wizard_shared_files` vs `wizard_shared_data` — different purposes!**

| What | Where | Format |
|------|-------|--------|
| Product images (main) | `wizard_shared_files.product_images` | `["url1", "url2"]` (array) |
| Logo | `wizard_shared_files.logo_image` | `"url"` (single string, NOT array) |
| Evidence/certs | `wizard_shared_files.evidence_images` | `["url1"]` (array) |
| Spokesperson faces | `wizard_shared_data.spokesperson_faces` | `["url1"]` (array) |
| Industry-specific images | `wizard_shared_data.{field}_images` | `["url1"]` (array) |

Example — uploading product images + logo:
```
mcp_tool_call("landing_ai_mcp", "update_session", {
  "user_token": token,
  "session_id": session_id,
  "data_json": "{\"wizard_shared_files\": {\"product_images\": [\"url1\", \"url2\"], \"logo_image\": \"url3\"}, \"wizard_shared_data\": {\"spokesperson_faces\": [\"url4\"]}}"
})
```

**Brand auto-enrichment**: When `create_session` is called with a brand that has
spokesperson/logo assets, they are auto-injected. In `update_session`, you must pass explicitly.

**Step 0: Determine industry category FIRST** — this decides which image fields to ask for.

The user's `industry_category` (set during brand creation) determines the relevant fields.
Only ask for images that match their industry. DO NOT show all 50+ fields.

**Always ask (all industries):**
- `product_images` — Main product/service photos
- `certification_images` — Awards, certifications (if any)

**Then ask ONLY the industry-specific fields:**

| industry_category | Additional fields to ask |
|-------------------|------------------------|
| `cosmetics` | `cosmetic_product_images`, `texture_images`, `before_after_images`, `ingredients_images`, `handheld_product_images`, `inner_packaging_images`, `outer_packaging_images` |
| `food`, `healthy_meals` | `harvest_images`, `farmer_story_images`, `handheld_produce_images`, `packaging_images`, `ingredients_images` |
| `supplements`, `biotech` | `biotech_lab_images`, `biotech_cert_images`, `biotech_product_images`, `ingredients_images`, `spec_sheet_images` |
| `restaurant` | `restaurant_exterior_images`, `restaurant_interior_images`, `dish_images`, `menu_images` |
| `medical_aesthetics` | `clinic_images`, `procedure_before_after_images`, `doctor_team_images`, `medical_cert_images` |
| `person`, `consultant` | `portrait_images`, `portfolio_images`, `event_speaking_images` |
| `film` | `film_still_images`, `poster_images`, `behind_scenes_images`, `cast_images` |
| `property`, `real_estate` | `property_exterior_images`, `property_interior_images`, `floor_plan_images`, `amenity_images`, `location_images` |
| `automotive` | `vehicle_exterior_images`, `vehicle_interior_images`, `vehicle_engine_images`, `vehicle_action_images` |
| `software`, `electronics` | `screenshot_images`, `mockup_images`, `device_angle_images`, `spec_sheet_images` |
| `education` | `screenshot_images`, `portfolio_images`, `event_speaking_images` |
| `fashion` | `handheld_product_images`, `texture_images`, `before_after_images`, `inner_packaging_images` |
| `general` (fallback) | `product_closeup_images`, `packaging_images`, `before_after_images` |

**Example conversation for `person` industry:**
```
Your brand is categorized as "person" (personal brand).
For the best LP, I need these images:

1. 📸 Portrait photo — professional headshot or graduation photo
2. 💼 Portfolio images — screenshots of your best projects
3. 🎤 Event/speaking images — conference talks, presentations (if any)

Which ones do you have? You can provide file paths or URLs.
```

### Regeneration (重新生成) — put URL into regenerate_stripe params

| Asset | Parameter | Example |
|-------|-----------|---------|
| Style reference | `reference_image_urls_json` | `'["https://...style.jpg"]'` |
| Text override | `mandatory_text_overrides_json` | `'{"headline": "New"}'` |

⚠️ Do NOT put regeneration reference images into `wizard_shared_files` — they go directly into `regenerate_stripe` params.

### Spokesperson — two valid paths

1. **`create_spokesperson` MCP tool** — creates a named spokesperson entity on the brand. Auto-injected on `create_session`.
2. **`wizard_shared_data.spokesperson_faces`** — direct URL array in session. Use when skipping brand spokesperson setup.

### Regeneration (重新生成) — put URL into regenerate_stripe

| Asset | Where to put public_url | How |
|-------|------------------------|-----|
| Reference/style image | `reference_image_urls_json` | `regenerate_stripe(reference_image_urls_json: '["url"]')` |
| Product correction | Same as above | Factory compares against reference photos |

⚠️ **DO NOT mix these up:**
- Wizard images → `update_session` with `wizard_shared_files`
- Regeneration images → `regenerate_stripe` with `reference_image_urls_json`
- Spokesperson → always `create_spokesperson`, never `wizard_shared_files`

**DO NOT ignore user photos.** If they give you a photo, it MUST be uploaded and used.
The LP Factory agent will incorporate the spokesperson into stripe images.

### Discovery Tips

- **Ask in batches of 3-4 questions** — don't overwhelm with 20 questions at once
- **Offer to read files**: if user provides a resume/CV, READ it and extract info
- **Offer URL scraping**: if they have a website, use `analyze_brand_url` to auto-fill
- **If user provides a photo** → immediately upload as spokesperson (see above)
- **Summarize back**: after gathering, confirm with user: "Here's what I have — anything to add?"
- **The more you gather, the better the LP** — spending 5 minutes here saves regeneration later

## Phase 3: Brand Creation

**Goal**: Find or create the user's brand profile.

### Step 2a: List existing brands
```
mcp_tool_call("landing_ai_mcp", "list_brands", { "user_token": token })
```

- If brands exist: present them and ask user to select one
- If no brands: proceed to brand creation

### Step 2b: Inspect brand completeness
```
mcp_tool_call("landing_ai_mcp", "get_brand", { "user_token": token, "brand_id": selected_brand_id })
```

Check for:
- Brand name ✓
- Brand description ✓
- Industry category ✓
- Primary color ✓
- Logo image ✓
- Product images (at least 1) ✓
- Value proposition ✓

### Step 2c: AI gap analysis
```
mcp_tool_call("landing_ai_mcp", "brand_gap_analysis", { "user_token": token, "brand_id": brand_id })
```

This returns a structured report of what's missing and recommendations.

## Phase 3: Fill Gaps

For each missing asset, offer solutions:

### Option A: Auto-scrape from website
```
mcp_tool_call("landing_ai_mcp", "analyze_brand_url", { "user_token": token, "url": "https://user-website.com" })
```
Extracts: logo, brand colors, product images, descriptions, social links.

### Option B: User uploads
Guide user to provide:
- Logo files (PNG/SVG preferred)
- Product photos (high-res, clean background preferred)
- Brand description text
- Product feature list

Upload via:
```
mcp_tool_call("landing_ai_mcp", "upload_brand_asset", {
  "user_token": token,
  "brand_id": brand_id,
  "asset_type": "logo" | "product_image" | "certification" | "spokesperson",
  "file_url": "https://..." | base64_data
})
```

### Option C: Create brand from scratch
```
mcp_tool_call("landing_ai_mcp", "create_brand", {
  "user_token": token,
  "name": "Brand Name",
  "description": "Brand description...",
  "industry": "cosmetics" | "biotech" | "health_food" | ...,
  "primary_color": "#2fa067"
})
```

## Phase 4: Readiness Assessment

After filling gaps, re-run gap analysis and score:

| Grade | Criteria | Action |
|-------|----------|--------|
| **A** (Ready) | Name + description + logo + 1+ product images + value prop | Proceed to Phase 2 (audience-target) |
| **B** (Usable) | Name + description + logo OR product images | Warn about quality impact, ask to proceed or improve |
| **C** (Incomplete) | Missing name or description or all images | Must fill more gaps before proceeding |

## Output

Present to user:
```
Brand: [brand_name]
Readiness: [A/B/C]
Assets: [count] images, [count] documents
Missing: [list of missing items, if any]
Brand ID: [brand_id]

→ Ready for audience targeting? [Yes / Let me add more assets]
```

Store `brand_id` and `user_token` for subsequent phases.

## Conversation Style

- Ask one question at a time — don't overwhelm
- If user has a website, always offer URL scraping first (fastest path)
- Explain WHY each asset matters: "Product images directly become the visual stripes in your landing page"
- Be encouraging: "Your brand looks great! Just need one more product image for the best results."
