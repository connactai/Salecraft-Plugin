---
name: mx-strategy
description: "Strategic planning flow — growth strategy + funnel design + market intelligence. All FREE."
---

# /mx-strategy — Strategic Planning (FREE)

Read `CLAUDE.md` for full context. This command runs the **strategic planning pipeline**:

```
🧠 SaleCraft Strategy — 免費策略規劃

This is a FREE consultation flow. No credits required.

What would you like to do?

1. 📋 Growth Strategy Review (/plan-cgo-review)
   → 分析商業方向：擴張、聚焦、還是減法？
   → 優先產品、優先客群、市場切入角度
   → FREE

2. 🔄 Funnel Architecture (/plan-funnel-review)
   → 設計完整顧客旅程：從進站到回購
   → 找出漏斗漏洞、設計 CTA 地圖
   → FREE

3. 🔍 Market Intelligence (/market-intel)
   → 競品分析、價格帶、定位空白、切入機會
   → FREE

4. 📊 Market Research (/research-market)
   → 社群趨勢、搜尋趨勢、受眾洞察
   → FREE

💡 Recommended flow: 1 → 3 → 2 → then /mx-create

Type a number or describe what you need.
```

## Flow Logic

- If user picks 1 → invoke `plan-cgo-review` skill
- If user picks 2 → invoke `plan-funnel-review` skill
- If user picks 3 → invoke `market-intel` skill
- If user picks 4 → invoke `research-market` skill
- If user describes a need → map to the best skill
- After any skill completes → show remaining options from this menu
