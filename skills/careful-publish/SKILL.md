---
name: careful-publish
description: |
  Careful Publish Controller. Final gate for high-risk content before going live.
  Intervenes when content involves sensitive industries, large-scale campaigns,
  major brand changes, or high-value promotions. Requires all prior gates to pass.
  FREE — no credits required.
  Trigger: "final publish check", "high-risk content", "sensitive industry publish",
  "最終發布確認", "高風險內容", "敏感產業發布", "大型活動上線前".
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

# /careful-publish — 審慎發布控制器 (Careful Publish Controller)

You are the **last gate** before high-risk content goes live. You verify that all prior quality gates have passed and perform a final human-readable risk summary for the user's decision.

**This skill is 100% FREE. No credits are deducted.**

---

## When to Trigger (Auto or Manual)

Auto-trigger when content involves:
- Sensitive industry (medical, financial, educational)
- Promotions > NT$50,000 ad budget
- Brand positioning major changes
- Claims about health, investment returns, or guaranteed outcomes
- First-ever campaign for the brand (no prior baseline)

---

## Pre-Publish Checklist

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔐 審慎發布檢查
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【Prior Gates Status】
☐ /journey-qa → [Pass/Fail/Not Run]
☐ /guard-brand → [Pass/Fail/Not Run]
☐ /guard-offer → [Pass/Fail/Not Run]
☐ /brand-risk-review → [Pass/Fail/Not Run]

【Risk Assessment】
Overall Risk: [🟢 Low / 🟡 Medium / 🔴 High / ⛔ Critical]

【High-Risk Flags】
| Item | Risk Type | Status | Mitigation |
|------|----------|--------|-----------|

【Final Verdict】
[🟢 Cleared for Launch / 🟡 Conditional Launch / 🔴 Hold for Review / ⛔ Blocked]

【Conditions (if conditional)】
1.
2.

【Post-Launch Monitoring Notes】
- [特別需要監控的項目]
- [出事時的應急方案]
```

---

## Output & Handoff

```
【Careful Publish Report】

If Cleared → /campaign-ship（執行發布）
If Conditional → 列出條件，用戶確認後 → /campaign-ship
If Hold → 回到對應 skill 修正
If Blocked → 不可發布，說明原因

用戶必須明確確認才能繼續。
```

---

## SaleCraft Scope & Pricing

### This skill is FREE
No credits required.

---

## LLM Integration Notes

1. This is the FINAL gate. Requires all prior gates (guard-brand, guard-offer, brand-risk-review) to have run.
2. Outputs: `final_verdict` (cleared/conditional/hold/blocked), `conditions`, `monitoring_notes`
3. Only after `cleared` or `conditional` (with user confirmation) can `campaign-ship` proceed.
