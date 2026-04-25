# Credit Calculator Reference

## How Credits Work

- Credits are paid in **pts** (`1 USD = 1 pt`, minimum top-up `$20 = 20 pts`)
- Credits are consumed on generation tools (`generate_session` / `regenerate_stripe` / `generate_ad` / `generate_carousel` / `generate_reels` etc.)
- Check balance: `get_me(user_token)` → `credits_remaining`

## Currency Rule (MANDATORY)

**The plugin uses ONE currency: USD (`$`). The only conversion is `$1 = 1 pt`. No other FX rate exists.**

- ✅ **All cost references in skills, examples, scripts, templates, and AI replies: USD (`$`) only**
- ❌ Never write **NTD / NT$ / TWD / 新台幣 / 台幣 / 新臺幣 / 臺幣** anywhere in plugin docs or AI replies
- ❌ Never write **EUR / € / GBP / £ / JPY / ¥ / 円 / CNY / RMB / 人民幣 / KRW / ₩ / 韓元 / 韓圜 / THB / ฿ / 銖 / VND / ₫ / 越南盾 / INR / ₹ / SGD / 新元 / HKD / 港幣 / 港元 / AUD / CAD / MYR / 令吉 / IDR / 盾 / 任何其他本地幣別**
- ❌ Never invent FX rates other than `$1 = 1 pt`. Even when the user asks 「1 pt 等於多少新台幣 / 日圓 / 人民幣」、回答只能限定在 `pts ↔ USD`、不要報外匯（「我這邊只用美元計算——1 pt = $1 USD、其他幣別請以你當下的銀行匯率為準」）
- 🔁 **User-input local currency**: 使用者用本地貨幣表達預算或產品價（「我月預算 50 萬日圓」/「我有 NT$30,000 廣告費」/「產品要賣 ¥299」）→ acknowledge their words but **always quote plugin's own cost in pts and USD only**（「好、那大約 X 美金、我這邊的 Y 功能是 Z pts ≈ $W USD」）。**不要**幫使用者把本地貨幣換算成 USD 報出實際數字（「50 萬日圓 ≈ 3,300 美金」這種匯率轉換不做、容易錯）
- 🎨 **Customer's own LP product prices** (rendered via `templates/sections/pricing-table.html` `{{this.currency}}` variable) are **out of scope of this rule** — those are *the customer's* prices for *their end customers*, set by them in any currency. Currency Rule applies only to **plugin's own cost reporting, examples, scripts, marketing-template placeholders, and AI replies** — not to user-supplied LP product prices
- 🔧 Backend `daily_budget` is **USD float** (`api-reference.md` §5) — never pass a `"currency"` field

This rule is enforced everywhere costs appear: this file, `CLAUDE.md` Pricing section, all SKILL.md cost references, all command files, all marketing-template examples (objection-handling scripts, VIP tier prices, ad-budget tiers, competitor price brackets, member upgrade paths).

## Landing Page Cost Formula

```
total_pts = 200 × requested_stripe_count × num_ta_groups
```

- `requested_stripe_count`: user-answered integer in 8-21 range (must be explicitly asked, never defaulted)
- `num_ta_groups`: number of selected target audiences (1-5 typical)
- `200`: per-page base cost, linear scaling

### Example Calculations

| Scenario | Pages | TAs | Total pts | ~USD |
|----------|-------|-----|-----------|------|
| Minimum LP | 8 | 1 | 1,600 | $53 |
| Standard LP | 10 | 1 | 2,000 | $67 |
| 12-page × 2 TA campaign | 12 | 2 | 4,800 | $160 |
| 15-page single TA | 15 | 1 | 3,000 | $100 |
| Full LP × 2 TA | 21 | 2 | 8,400 | $280 |

### Pre-deduct + adjustment mechanism

- `generate_session` **pre-deducts** `requested_stripe_count × 200` per TA at start
- After generation:
  - `actual_stripe_count >= requested` → no extra charge (free overdelivery)
  - `actual_stripe_count < requested` → `stripe_adjustment` refunds difference
- Net cost always = `actual_stripe_count × 200` per TA

## Pre-Generation Check

Before calling `generate_session`:

1. Call `get_me(user_token)` to get `credits_remaining`
2. Calculate `total_pts` using the formula above
3. If `credits_remaining < total_pts`:
   - Inform user with exact math: "Need X pts but only have Y remaining"
   - Suggest reducing TAs or page count
   - Do NOT proceed with generation

## Post-Generation Edit Tool Cost Reference

**🔴 Critical rule**: Never extrapolate a tool's cost from another tool's price. If it's not explicitly listed below, **check `get_generation_settings` or run a preflight** (e.g. `seo_preflight`) before quoting a cost to the user. See the anti-pattern in CLAUDE.md rule 6.5 ("cost extrapolation from tool name").

### FREE — config-only edits (no regeneration, no AI pipeline re-run)

| Tool | What it does |
|------|-----|
| `update_stripe_text` / `update_stripe_texts` | Replace text on a stripe |
| `update_stripe_text_styling` / `update_stripe_text_boxes` | Font / size / color / position of text |
| `update_cta` / `update_cta_link` / `update_cta_style` | CTA button text / URL / style |
| `set_stripe_overlay` | Semi-transparent color layer for text readability |
| `set_stripe_soft_edge` | Gradient fade between stripes |
| `crop_stripe` / `reset_crop` | Change visible rectangle of the image (no re-render) |
| `update_stripe_background` | Change background color / image |
| `hide_stripe` / `restore_stripe` | Toggle visibility |
| `reorder_stripes` | Change stripe order |
| `update_image_layers` | Swap uploaded images into image layers |
| `upload_logo` | Change the header logo at top-left |
| `patch_landing_config` | Generic config patch (header_nav / footer / primary_color / …) |
| `mask_stripe` / `edit_regions` | Region-specific image edits (may have paid variants — verify via response) |

### PAID — actually re-runs AI pipeline

| Tool | Cost |
|------|-----|
| `regenerate_stripe` | 100 pts / stripe（re-runs Factory Agent on that stripe）|
| `regenerate_scene` (reels) | check response `credits_deducted` |
| `regenerate_project_stripe` / `regenerate_campaign_stripe` | check response |
| `run_seo_optimize` | 500 pts（beta: free、always `seo_preflight` first to confirm actual cost） |

### VERIFY BEFORE QUOTING — unknown / conditional cost

| Tool | How to check |
|------|-----|
| `generate_ad` | response includes `credits_deducted` |
| `generate_carousel` | response includes `credits_deducted` |
| `generate_reels` | 100 pts / sec（confirmed）|
| Any new tool not listed above | call once in preview mode if available, OR ask backend team, OR tell user "this action may deduct credits — let me check before I run it" |

## Ad Creative Generation

- `generate_ad` — may consume additional credits
- Check cost via `get_ad_result` response
