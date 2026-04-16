---
name: mx-engage
description: "Engagement + conversion strategy — interaction design, closing scripts, objection handling. All FREE."
---

# /mx-engage — Engagement & Conversion (FREE)

Read `CLAUDE.md` for full context. This command runs the **engagement and conversion pipeline**:

```
💬 SaleCraft Engage & Convert — 免費互動與成交策略

This is a FREE consultation flow. No credits required.

What would you like to do?

1. 💬 Engagement Strategy (/engage-operator)
   → 私訊腳本、FAQ 對答樹、自動回覆、留資引導、預約腳本
   → FREE

2. 🎯 Conversion Strategy (/conversion-closer)
   → 異議處理庫、價格鋪墊、社會證明、收單腳本、跟進節奏
   → FREE

3. 🔍 Journey QA (/journey-qa)
   → 測試完整顧客旅程（從進站到成交）
   → FREE

💡 Recommended flow: 1 → 2 → 3 → then /mx-audit

Type a number or describe your situation.
```

## Flow Logic

- If user picks 1 → invoke `engage-operator` skill
- If user picks 2 → invoke `conversion-closer` skill
- If user picks 3 → invoke `journey-qa` skill
- If user says "people visit but don't buy" → start with `conversion-closer`
- If user says "no one contacts us" → start with `engage-operator`
- After completion → suggest the next skill in sequence
