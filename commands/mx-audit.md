---
name: mx-audit
description: "Quality & governance audit — brand, offer, compliance, journey QA, launch checklist. All FREE."
---

# /mx-audit — Quality & Governance Audit (FREE)

Read `CLAUDE.md` for full context. This command runs the **quality and governance pipeline**:

```
🛡️ SaleCraft Audit — 免費品質與治理檢查

This is a FREE audit flow. No credits required.

What would you like to check?

1. 🛡️ Brand Consistency (/guard-brand)
   → 品牌語氣、視覺調性、禁用詞、差異化一致性
   → FREE

2. 💰 Offer Consistency (/guard-offer)
   → 價格、承諾、優惠條件跨觸點一致性
   → FREE

3. ⚖️ Compliance Review (/brand-risk-review)
   → 療效承諾、投資保證、過度宣稱、法規風險
   → FREE

4. 🔍 Journey QA (/journey-qa)
   → 測試完整顧客旅程（頁面、CTA、表單、手機）
   → FREE

5. 🔐 Careful Publish (/careful-publish)
   → 高風險內容最終閘門（敏感產業必跑）
   → FREE

6. 🚀 Launch Checklist (/campaign-ship)
   → 上線前完整檢查清單
   → FREE

💡 Recommended flow (full audit):
   1 → 2 → 3 → 4 → 5 → 6

   Quick audit (before launch):
   4 → 6

Type a number or describe what you want to check.
```

## Flow Logic

- Full audit: run skills in sequence 1 → 2 → 3 → 4 → 5 → 6
- Quick audit: just `journey-qa` → `campaign-ship`
- If sensitive industry detected → auto-include `brand-risk-review` + `careful-publish`
- Each skill outputs a pass/fail → aggregate into final launch readiness
