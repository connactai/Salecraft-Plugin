# MarketingX — Complete Workflow Reference

## End-to-End Flow

```
User Request
  │
  ▼
Phase 1: Brand Onboarding [brand-onboard]
  │  Authenticate → Check brand → Verify assets → Fill gaps
  │  Output: brand_id + readiness_score
  ▼
Phase 2: Audience Targeting [audience-target]
  │  AI suggest TAs → User selects → Aspect ratio → Credit estimate → Confirm
  │  Output: ta_config + cost_confirmed
  ▼
Phase 3: LP Generation [generate-landing]
  │  Create session → Trigger pipeline → Poll status → Quality check
  │  Output: campaign_id + preview
  ▼
Phase 4: Editing [edit-landing] (iterative)
  │  User describes changes → Map to MCP tools → Execute → Preview → Loop
  │  Output: finalized campaign_id
  ▼
Phase 5: Homepage Building [homepage-builder]
  │  Select template → Embed LP stripes → Apply i18n → Generate files
  │  Output: deployable HTML/CSS/JS
  ▼
Phase 6a: Social Publishing [publish-social]
  │  Select platforms → Import content → Schedule → Publish
  │  Output: post_urls
  ▼
Phase 6b: Ad Campaigns [publish-ads]
  │  Select objective → Generate creative → Set budget → Create campaign
  │  Output: campaign_ids + monitoring links
```

## Phase Details

### Phase 1: Brand Onboarding

**Goal**: Ensure sufficient brand assets exist before generation.

**Steps**:
1. Authenticate: `login(email, password)` → store `user_token`
2. Check brands: `list_brands(user_token)` → show existing brands
3. If brand exists: `get_brand(user_token, brand_id)` → check completeness
4. Run gap analysis: `brand_gap_analysis(user_token, brand_id)` → identify missing items
5. Fill gaps:
   - Auto-scrape: `analyze_brand_url(user_token, url)` — extract from website
   - Manual upload: `upload_brand_asset(user_token, brand_id, ...)` — user provides files
6. Output: `brand_id` with readiness grade (A = ready, B = usable, C = needs more)

**Minimum assets for quality generation**:
- Brand name + description
- At least 1 logo image
- At least 1 product image
- Industry category
- Value proposition (1-2 sentences)

### Phase 2: Audience Targeting

**Goal**: Define who the LP is for, how many pages, and confirm cost.

**Steps**:
1. Generate TA options: `generate_ta_options(user_token, brand_name, description, industry_category, ...)` → 3-5 AI-suggested audiences
2. User selects TAs (can customize or add their own)
3. Select aspect ratio: 16:9 (landscape), 9:16 (portrait), or both
4. Calculate page count: `pages = TAs × aspect_ratios × locales`
5. Check balance: `get_me(user_token)` → remaining credits
6. Estimate cost: pages × per_page_credit_cost
7. Present summary and confirm with user

**Optional market research** (before TA selection):
- `google_trends_mcp` — validate TA with search trend data
- `x_mcp` / `reddit_mcp` — discover audience pain points
- `tiktok_mcp` — identify trending content for younger audiences

### Phase 3: LP Generation

**Goal**: Generate landing pages through the AI agent pipeline.

**Steps**:
1. Create session: `create_session(user_token, session_name, brand_name, product_name, base_description)`
2. Configure: `update_session(user_token, session_id, ...)` — set TA, aspect ratio, locale
3. Trigger generation: `generate_session(user_token, session_id)`
4. Poll until complete (every 5s, max 60 polls):
   ```
   get_session(user_token, session_id) → check status field
   - "generating" → continue polling, report progress
   - "complete" → proceed to step 5
   - "error" → report error, suggest retry
   ```
5. Retrieve stripes: `list_stripes(user_token, session_id)` → verify all generated
6. Quality check: inspect each stripe for completeness
7. **MANDATORY: Provide preview links to the user**:
   - Call `create_share_token(user_token, campaign_id)` → get share_token
   - Call `get_public_landing_page(user_token, campaign_id)` → get stitched_image_url + per-stripe URLs
   - Present ALL links to the user (full LP image + each stripe image)
   - User MUST see the LP before deciding to edit or proceed
8. If generating both aspect ratios: repeat steps 1-7 for the second ratio

**Progress reporting**: Show which agent is currently active (Strategist → Architect → Factory → Reflector).

### Phase 4: Editing

**Goal**: Refine the generated LP based on user feedback.

**Intent mapping** — translate natural language to MCP tools:

| User says | Tool to call |
|-----------|-------------|
| "Change the headline to..." | `update_stripe_texts` |
| "Make the text bigger/bolder" | `update_stripe_text_styling` |
| "Use a different background" | `update_stripe_background` |
| "Replace the product image" | `update_image_layers` |
| "Add a dark overlay" | `set_stripe_overlay` |
| "Add a gradient edge" | `set_stripe_soft_edge` |
| "Crop this stripe" | `crop_stripe` |
| "Redo this stripe completely" | `regenerate_stripe` |
| "Undo that change" | `undo_stripe` |
| "Move this stripe up/down" | `reorder_stripes` |
| "Hide this stripe" | `hide_stripe` |
| "Update SEO title/description" | `update_seo` |
| "Edit the FAQ section" | `update_faq_content` |

**Editing loop**: After each edit, offer to show preview. Continue until user says "done" or "looks good".

### Phase 5: Homepage Building

**Goal**: Create a deployable website homepage embedding the generated LP.

**This is the main new component.** The homepage-builder skill generates HTML/CSS/JS files.

**Steps**:
1. Get LP data: `get_landing_page(user_token, campaign_id)` → full LP config
2. Choose embedding strategy:
   - **Image Embed** (default, better performance): `get_export_image_url(user_token, campaign_id, stripe_idx)` per stripe
   - **iframe Embed** (interactive): `export_html(user_token, campaign_id)` for full HTML
3. User selects homepage layout: single-product / multi-product / campaign
4. Compose sections from `templates/sections/`:
   - hero-with-lp.html — First stripe as hero banner
   - features-grid.html — Feature highlights
   - testimonials.html — Social proof
   - pricing-table.html — Pricing (if applicable)
   - cta-banner.html — Call-to-action
   - footer.html — Footer with links
5. Apply i18n from `templates/i18n/{locale}.json`
6. Apply aspect ratio CSS from `templates/embed/aspect-ratio.css`
7. Output directory:
   ```
   output/
   ├── index.html
   ├── styles.css
   ├── script.js
   └── assets/
       ├── stripe-0.webp
       ├── stripe-1.webp
       └── ...
   ```

### Phase 6a: Social Publishing

**Goal**: Post LP content to social media platforms.

**Steps**:
1. List connected accounts: `list_accounts(user_token)` via `zereo_social_mcp`
2. Import LP content: `import_from_session(user_token, session_id)`
3. Validate media: `validate_content_media(user_token, ...)`
4. Get optimal times: `suggest_schedule(user_token)`
5. Publish: `publish_multi(user_token, targets_json)` — targets = list of {platform, account_id, content}

### Phase 6b: Ad Campaigns

**Goal**: Create and launch ad campaigns on Meta / Google.

**Steps**:
1. Verify ad accounts: `list_accounts(user_token)` → filter for ad-capable accounts
2. Check permissions: `get_account_capability(user_token, account_id)`
3. Present objectives: `get_ad_objectives(user_token)` — awareness / traffic / conversions
4. Generate ad creative: `generate_ad(user_token, session_id, platform)` via `landing_ai_mcp`
5. Wait for creative: `get_ad_result(user_token, task_id)` — poll until ready
6. User sets: budget, schedule, targeting
7. Create campaign: `create_ad_campaign(user_token, data_json)` via `zereo_social_mcp`
8. Monitor: `get_ad_campaign(user_token, campaign_id)` — report status

## Cross-Phase State

These values carry across phases — never re-ask for something already known:

```
{
  user_token: "eyJ...",          // Phase 1
  brand_id: "brand_abc123",      // Phase 1
  ta_groups: [...],              // Phase 2
  aspect_ratio: "16:9",          // Phase 2
  locale: "zh-TW",              // Phase 2
  session_id: "sess_xyz789",     // Phase 3
  campaign_id: "camp_def456",    // Phase 3
  landing_page_id: "lp_ghi012", // Phase 3
  homepage_files: ["index.html", ...], // Phase 5
  ad_campaign_ids: [...]         // Phase 6b
}
```
