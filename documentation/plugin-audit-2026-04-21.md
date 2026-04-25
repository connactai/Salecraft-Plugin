# Plugin-Wide Audit Report — 2026-04-21

Audit scope: 4 categories of "content integrity" claims in the plugin (platform-lock claims, fake pricing, fake features, skill count consistency). Wave 1 covers Audits 1 / 2 / 4. Audits 3 and 5 deferred to Wave 2 pending user review of Wave 1.

---

## Audit 1 — 平台鎖 claim

**Status: ✅ PASS**

Grep patterns scanned (English + Chinese):
- `only (work|run|available|supported)`
- `requires.*claude.*code`
- `must (install|have|use).*(claude code|chatgpt|gemini)`
- `this (only|just).*(works|runs) (on|with|in)`
- `platform.specific` / `plugin.only` / `claude.code.only`
- 僅 / 只能 / 此功能僅 / 專為.*Claude Code

**All matches are anti-violation rules** (explicit instructions telling the LLM NOT to platform-lock):

| File | Line | Content |
|---|---|---|
| CLAUDE.md | 665 | "DO NOT tell users they need to install Claude Code... Almost every modern LLM can run the consultation skills as instructions." |
| CLAUDE.md | 1637 | "Platform agnostic — SaleCraft works on ANY AI platform (ChatGPT, Claude, Gemini, Kimi, GLM, OpenClaw, etc.). Never say 'this only works on [platform]'..." |
| commands/salecraft-audit.md | 49 | "SaleCraft runs on any AI platform. Never say 'this only works on [specific platform].'" |
| commands/salecraft-engage.md | 41 | same message |
| commands/salecraft-retain.md | 41 | same message |
| commands/salecraft-strategy.md | 46 | same message |
| commands/salecraft.md | 68, 80 | same message + "there is no installation, ever" |
| prompts/BOOTSTRAP.md | 5 | "SaleCraft works on any AI platform. There is no installation..." |

**Edge case reviewed**:
- CLAUDE.md L659: "This repo is structurally a Claude Code plugin (.claude-plugin/ manifest, commands/ slash commands, skills/ Anthropic-format skills, root CLAUDE.md)" — factual description of repo file layout. Same paragraph immediately clarifies: "When read by other LLMs... this repo functions as an instruction set". **Not a violation** — describing structure is not claiming exclusivity.

**Fix needed**: None.

---

## Audit 2 — 虛假定價

**Status: 🟠 HAS ISSUES** — 1 critical drift + 2 unverified numbers

### Ground truth sources (confirmed consistent)

| Pricing claim | Ground truth | Status |
|---|---|---|
| 1 USD = 1 pt / Min $20 = 20 pts | All sources agree | ✅ |
| LP generation: 200 pts/page × page_count × num_tas | CLAUDE.md pricing table, README, commands/salecraft-create | ✅ |
| Regenerate single stripe: 100 pts | Multiple sources agree | ✅ |
| Quick Ad (single image): 200 pts | Multiple sources agree | ✅ |
| Carousel: 300 + 100×N pts | Consistent formula | ✅ |
| Social Copy: 100 pts/set | Consistent | ✅ |
| Reels: 100 pts/sec (min 5s, max 60s) | Consistent | ✅ |
| SEO optimization: 500 pts (beta free) | Consistent + seo_preflight gate | ✅ |
| QR Code: 5 pts | Mentioned only in pricing table | ✅ |
| `generate_ta_spokesperson`: 0 pts (free quota) | Consolidated across all sources | ✅ |

### 🔴 Critical drift

**`lib/credit-calculator.md` L10-28** uses an **outdated "credits" pricing model** that contradicts all other documentation:

```
total_credits = num_ta_groups × num_aspect_ratios × credits_per_page
```

Examples in that block claim "1 credit" / "2 credits" / "3 credits" per LP — which is nonsense under the current 200-pts-per-page model.

The **same file's bottom section** (L41-85) uses the correct pts model. The top section should be purged.

**Impact**: LLM reading this file sees two contradictory cost models in one document. Depending on which block it references first, could quote 1 credit or 200 pts for the same action.

**Fix**:
- Replace L10-28 with current 200-pts/page formula
- Or delete that block entirely since the bottom section + README/CLAUDE.md pricing tables already cover it

### ⚠️ Unverified numbers

1. **`skills/audience-target/SKILL.md:1042`**: "This skill costs 5-15 pts per TA generation."
   - `generate_ta_options` returns candidates without user credits being deducted (per backend source)
   - The "5-15 pts" figure has no ground truth reference
   - **Recommendation**: Verify with backend or remove this line. Likely wrong (TA candidate generation is free).

2. **`skills/generate-reels/SKILL.md:400`**: "| Scene regeneration | ~50 pts |"
   - No other source documents scene regeneration cost
   - Backend `regenerate_scene` response likely carries `credits_deducted`; plugin should reference that instead of hardcoding ~50 pts
   - **Recommendation**: Replace with "check response `credits_deducted`" language, matching other tools in `lib/credit-calculator.md` VERIFY section.

### 🟡 Minor issues

- **"stripe" vs "page" used interchangeably** in several places (e.g., `skills/generate-landing/SKILL.md:1190` says "200 pts/stripe"). Technical: both are the same thing in backend, but user-facing communication should standardize on "page" per JARGON BLACKLIST.
- **CLAUDE.md Pricing table** (L1599-1609) intentionally duplicates content from `lib/credit-calculator.md`. This is deliberate (quick-reference) but creates drift risk. Already flagged in v3 CLAUDE.md cleanup plan.

### Fix summary

| Priority | File | Fix |
|---|---|---|
| 🔴 P0 | `lib/credit-calculator.md` L10-28 | Delete or replace outdated "credits" model |
| 🟠 P1 | `skills/audience-target/SKILL.md:1042` | Verify or remove "5-15 pts per TA" |
| 🟠 P1 | `skills/generate-reels/SKILL.md:400` | Replace "~50 pts" with "check credits_deducted" |
| 🟡 P2 | Multiple files | Standardize "page" vs "stripe" in user-facing pricing descriptions |

---

## Audit 4 — Skill 數量一致性

**Status: 🟠 HAS ISSUES** — CLAUDE.md off-by-one, README incomplete

### Ground truth

`skills/` directory contains **26 skills**:

```
audience-target, brand-memory, brand-onboard, brand-risk-review,
campaign-ship, careful-publish, conversion-closer, document-release,
edit-landing, engage-operator, generate-landing, generate-reels,
growth-retro, guard-brand, guard-offer, homepage-builder, journey-qa,
market-intel, member-lifecycle, plan-cgo-review, plan-funnel-review,
publish-ads, publish-social, research-market, saleskit, upload-media
```

### Discrepancies

| File | Claimed count | Actually lists | Missing skills |
|---|---|---|---|
| `skills/` directory | — | **26** | (ground truth) |
| `CLAUDE.md` L1435 header: "Available Skills (25)" | 25 | 25 | `upload-media` |
| `README.md` L80 header: "Skills (26)" | 26 (header correct) | 24 (body incomplete) | `brand-memory`, `upload-media` |

### Fix summary

| File | Action |
|---|---|
| `CLAUDE.md` L1435 | Change "Available Skills (25)" → "(26)"; add `upload-media` to appropriate tier |
| `README.md` L80 region | Add `brand-memory` + `upload-media` to skill tables |

### Note on missing skills

- **`brand-memory`** (CLAUDE.md already has it at L1495): "Auto-record files, prompts, and metadata per brand for personalized experience" — runs silently in background. README should mention it even if as "background skill, no user interaction needed".
- **`upload-media`** (exists as directory): Needs a line confirming what it does and when triggered. If it's an internal utility skill, both CLAUDE.md and README should document it.

---

## Wave 1 Summary

| Audit | Status | Issues found |
|---|---|---|
| 1. Platform lock | ✅ PASS | 0 violations |
| 2. Fake pricing | 🟠 1 critical + 2 unverified + minor stripe/page drift | 4 fixes |
| 4. Skill count | 🟠 Off-by-one in CLAUDE.md, 2 skills missing from README | 2 fixes |

**Critical (P0)**: 1 item (outdated credits model in `lib/credit-calculator.md`)
**High (P1)**: 3 items (2 unverified prices, skill count fixes)
**Minor (P2)**: 1 item (stripe/page standardization)

Total fixes: **~6 distinct edits**, all low-risk and localized. No file structure changes needed, no SKILL behavior changes needed.

---

## Wave 2 Preview (deferred pending user review)

**Audit 3** — Virtual features: each plugin claim must have corresponding skill + MCP tool + command. Previously caught `i18n-adapt` fabrication and `get_suggested_buttons` misuse. Need to walk through remaining feature claims in README Features section and SKILL description frontmatters.

**Audit 5** — Backend coverage: plugin references 74/311 landing_ai_mcp tools. Of the 237 unreferenced, some are intentionally internal (backfill_gcs_urls, etc.) but others may be user-facing features that plugin doesn't surface. Requires category-by-category sampling.

---

## Next step (recommended)

1. User reviews this report
2. If Wave 1 findings accepted, apply 6 fixes as a single commit (all localized, low-risk)
3. After Wave 1 fixes land and validate, decide whether to run Wave 2 (Audit 3 + 5)

---

# Wave 2 — Fake Features & Backend Coverage

## Audit 3 — 虛假功能（feature claim vs 實作）

**Status: 🔴 HAS ISSUES** — 1 major fabrication found

### Methodology

Walked through major feature claims in README Features section, CLAUDE.md, and relevant SKILL docs. Cross-checked each against actual backend MCP tools in `Service_system/landing_ai_mcp/tools/` and `Service_system/zereo_social_mcp/tools/`.

### Findings

| Feature claim | Source | Backend tool | Status |
|---|---|---|---|
| 15+ languages (en, zh-TW, zh-CN, ja, ko, vi, fr, th, es, pt, ar, de, id, ms, hi) | README L145 | `templates/i18n/` has 15 JSON locale files | ✅ Verified |
| AI Landing Pages (30-min turnaround) | README L147 | `generate_session` + Strategist/Architect/Factory pipeline | ✅ Verified |
| Social publishing — IG, FB, TikTok one-click | README L148 | `publish_post` supports `ig_post/ig_story/ig_reel/fb_post/fb_story/tt_video` | ✅ Verified |
| **Ad campaigns — Meta + Google Ads creation** | **README L149** | `create_ad_campaign` is **Meta-only** — no Google Ads creation tool exists | 🔴 **FABRICATED** |
| TikTok sandbox mode posting | publish-social L62 | `tt_video` post type supported, sandbox caveat documented | ✅ Verified |
| Homepage builder | homepage-builder SKILL | Composed from `export_html` + `download_stripe` (both real) | ✅ Verified |
| Reel promotion (IG/FB boost) | publish-ads L286 | `promote_reel` exists in `zereo_social_mcp/tools/ads.py` | ✅ Verified |
| QR Code generation | multiple skills | `generate_qr`, `generate_qr_card`, `composite_qr` all exist in zereo_social_mcp | ✅ Verified |
| Google Drive asset import | CLAUDE.md + README | 7 `gdrive_*` tools exist in landing_ai_mcp/content.py | ✅ Backend exists (see Audit 5 re: surface) |
| Spokesperson AI generation | audience-target SKILL | `generate_ta_spokesperson` + `get_spokesperson_generation_status` | ✅ Verified |

### 🔴 Critical fabrication — Google Ads creation

**Claimed at**:
- `README.md:149` — "Ad campaigns — Meta + Google Ads creation"
- `skills/publish-ads/SKILL.md:61` — "🔍 Google Ads — ACME Search (can create campaigns)"
- `skills/publish-ads/SKILL.md:281` — "Create Google Ads campaign too"
- `lib/ad-platform-specs.md:34-59` — Full Google Ads campaign types + asset specs section
- `CLAUDE.md:1708` — "Ad Campaigns (11 tools) — Meta/Google ads"

**Backend reality**:
- `zereo_social_mcp/tools/ads.py`'s `create_ad_campaign` docstring: **"Create a new Meta (Facebook/Instagram) ad campaign"**
- grep for `google_ads|googleads|adwords` in backend tool definitions: **0 matches**
- No Google Ads MCP tool exists

**Impact**: User reads README, asks LLM to "create Google Ads campaign", LLM has no backend path. Result: either (a) silently pretends, (b) tries `create_ad_campaign` and creates a Meta campaign by mistake, or (c) apologizes but damage to user trust is done.

**Note on aspect_ratio references**: Mentions like "16:9 適合 Google Ads" (in CLAUDE.md L372, generate-landing L379/467, commands/salecraft-create L100) are **not fabrications** — they advise aspect ratios users might need IF they later export the image to Google Ads manually. These don't claim plugin creates Google Ads campaigns. Keep as-is.

### Fix summary (Audit 3)

| Priority | File | Fix |
|---|---|---|
| 🔴 P0 | `README.md:149` | "Meta + Google Ads creation" → "Meta (Facebook/Instagram) campaigns + ad creative export for other platforms" |
| 🔴 P0 | `skills/publish-ads/SKILL.md:61` | Remove "Google Ads" from ad-capable accounts list, OR mark as "creative export only, no campaign creation" |
| 🔴 P0 | `skills/publish-ads/SKILL.md:281` | Remove "A) Create Google Ads campaign too" (no such capability) |
| 🔴 P0 | `lib/ad-platform-specs.md:34-59` | Mark Google Ads section as "creative specs reference (for manual export only — plugin does not create Google Ads campaigns)" OR delete section |
| 🔴 P0 | `CLAUDE.md:1708` | "Meta/Google ads" → "Meta ads" |

---

## Audit 5 — Backend 覆蓋率（211 未覆蓋工具）

**Status: ⚠️ INTENTIONAL GAPS + 2 SURFACING OPPORTUNITIES**

### Backend tool count per category

| Category | Tools | Notes |
|---|---|---|
| landing_pages | 56 | LP editing + stripe ops |
| content | 50 | Mixed: brand/campaign/gdrive/pdf/qr/memory |
| sessions | 34 | Session lifecycle + generation |
| brands | 32 | Brand/spokesperson/analysis |
| reels | 26 | Reels generation + templates |
| conversations | 20 | Chat/messaging/phase data |
| auth | 15 | Auth + account mgmt |
| todos | 14 | Todo system |
| prediction | 13 | AI prediction tools |
| workspace, tracking, campaigns_extended | 10 each | Admin/analytics |
| Other smaller | 12 | ratings/pricing/analytics/demo/legacy/spokespersons_account |
| **Total** | **311** | |

Plugin references: 74 unique landing_ai_mcp tools + zereo_social_mcp tools. **Gap: ~237 tools not surfaced**.

### Gap classification

Most of the 237 are intentional:
- **CRUD primitives** (delete_*, list_*, get_* for internal state): plugin abstracts these behind higher-level skills
- **Admin/debug tools** (`backfill_gcs_urls`, `mcp_servers_list`): not user-facing
- **Legacy tools** (`legacy_generation.py` 6 tools): explicitly deprecated
- **Demo/tracking**: non-user-facing
- **Prediction**: analytics backend, not flow-path
- **Spokespersons_account** (6 tools): advanced spokesperson management, plugin covers the essentials via brand-level create_spokesperson

### 🟠 Real gaps — user-facing features plugin doesn't surface

#### Gap 1: Reels template system (4 tools not surfaced)

Backend has:
- `list_templates` — browse pre-made reel templates
- `apply_template` — apply a template to a session
- `save_as_template` — save current reel as reusable template
- `ai_suggest_reel` — AI suggests reel concept

Plugin `skills/generate-reels/SKILL.md` **never mentions templates**. User has to start every reel from scratch even though backend supports templates.

**Recommendation**: Add Phase 0 "Pick a template?" to generate-reels SKILL, listing `list_templates` → `apply_template` flow.

**Severity**: 🟠 Medium — feature exists but is dark to LLM → users miss efficiency boost

#### Gap 2: Scene-level reel editing (3 tools not surfaced)

Backend has:
- `list_scenes` — list all scenes in a reel
- `get_scene` — get scene detail
- `update_scene` — edit a specific scene

Plugin mentions `regenerate_scene` but not `update_scene` (text/config changes vs full regen). User who wants to tweak one line of a scene has to regenerate the whole thing.

**Recommendation**: Add scene-editing subsection to generate-reels SKILL with update_scene as the "edit without regen" path.

**Severity**: 🟠 Medium

#### Gap 3: `ai_suggest_reel` never surfaced (covered by Gap 1 but worth calling out)

When user says "I don't know what reel to make", there's a direct AI-suggest flow that plugin ignores entirely.

**Severity**: 🟡 Low — LLM can substitute via chat

#### Not-really-gaps (already covered or intentionally hidden)

- **Google Drive** (7 tools): CLAUDE.md L1330-1332 documents Google Drive binding and `gdrive_import_shared_link`. Coverage is thin (only shared-link import mentioned, not full Drive API flow) but the user-visible flow (bind account on marketingx page → LLM imports via shared link) is documented. Marginally under-documented but not a gap.
- **`import_pdf`**: brand-onboard mentions PDF upload but doesn't reference this specific MCP tool. Implementation uses it implicitly.
- **`compress_memory` / `delete_memory`**: brand-memory SKILL handles these silently, no user surfacing needed.
- **`summarize_conversation`**: could be a feature but not a claimed one — no drift, just unused capability.

### Fix summary (Audit 5)

| Priority | Gap | Proposed action |
|---|---|---|
| 🟠 P1 | Reels templates (`list_templates` / `apply_template` / `save_as_template`) | Add Phase 0 in generate-reels SKILL: "Start from a template?" |
| 🟠 P1 | Scene-level edits (`list_scenes` / `get_scene` / `update_scene`) | Add scene-edit subsection in generate-reels (edit-without-regen path) |
| 🟡 P2 | `ai_suggest_reel` | Cross-reference from saleskit ("if user doesn't know what reel to make") |

---

## Wave 2 Summary

| Audit | Status | Critical issues |
|---|---|---|
| 3. Fake features | 🔴 HAS ISSUES | Google Ads creation fabricated (5 file:line refs) |
| 5. Backend coverage | ⚠️ PARTIAL GAPS | 2 P1 opportunities in Reels (templates + scene edits), 1 P2 (ai_suggest_reel) |

**P0 fixes (Audit 3)**: 5 edits across README/publish-ads/lib/CLAUDE.md to remove Google Ads creation claims
**P1 additions (Audit 5)**: 2 new sections in generate-reels SKILL (optional — improves but doesn't fix broken)

---

## Combined Wave 1 + Wave 2 Fix Roadmap

**Done in Wave 1 commit**: P0 credits model + P1 audience-target/generate-reels pts + skill count

**Remaining (this commit)**:
- Audit 3 P0: Remove Google Ads creation claims (5 edits)
- Audit 5 P1: Add Reels template + scene-edit sections (2 new subsections)
- Audit 5 P2: ai_suggest_reel cross-reference (1 edit)

