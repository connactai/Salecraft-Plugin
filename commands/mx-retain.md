---
name: mx-retain
description: "Retention + growth loop — member lifecycle, referral programs, performance review. All FREE."
---

# /mx-retain — Retention & Growth Loop (FREE)

Read `CLAUDE.md` for full context. This command runs the **retention and growth review pipeline**:

```
🔄 SaleCraft Retain & Grow — 免費會員經營與成長回顧

This is a FREE consultation flow. No credits required.

What would you like to do?

1. 🔄 Member Lifecycle (/member-lifecycle)
   → 回購觸發、推薦方案、VIP 制度、客戶升級路徑
   → FREE

2. 📊 Growth Retrospective (/growth-retro)
   → 活動成效回顧、KPI 分析、下一輪優化假設
   → FREE

3. 📄 Document & Compile (/document-release)
   → 整理話術手冊、FAQ 文件、案例彙編、活動報告
   → FREE

💡 Recommended flow: 1 → 2 → 3 → then new sprint via /saleskit

Type a number or describe what you need.
```

## Flow Logic

- If user picks 1 → invoke `member-lifecycle` skill
- If user picks 2 → invoke `growth-retro` skill
- If user picks 3 → invoke `document-release` skill
- If user says "customers don't come back" → start with `member-lifecycle`
- If user says "how did the campaign go" → start with `growth-retro`
- After `growth-retro` → suggest new sprint cycle via `/saleskit`
