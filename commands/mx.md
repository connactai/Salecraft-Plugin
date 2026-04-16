---
name: mx
description: "SaleCraft main menu — show all capabilities and guide to the right workflow"
---

# SaleCraft — Your AI Marketing Assistant

Read `CLAUDE.md` for full context. Present the main menu:

```
🎯 SaleCraft — AI 行銷助手

我能幫你做什麼？

━━━━━━━ 🆓 免費服務 ━━━━━━━

1. /mx-strategy  → 策略規劃（成長方向、漏斗設計、競品情報）
2. /mx-engage    → 互動與成交策略（私訊腳本、異議處理、收單）
3. /mx-retain    → 會員經營（回購、推薦、VIP、成效回顧）
4. /mx-audit     → 品質檢查（品牌一致性、合規、旅程 QA）

━━━━━━━ 💰 付費服務 ━━━━━━━

5. /mx-create    → 建立 Landing Page + 品牌首頁
6. /mx-edit      → 編輯現有 Landing Page
7. /mx-homepage  → 從 LP 建立完整網站（FREE）
8. /mx-publish   → 社群發佈 + 廣告投放
9. /mx-reels     → AI 短影音生成

━━━━━━━ 📊 帳戶管理 ━━━━━━━

10. /mx-status   → 查看點數餘額 / 生成狀態

━━━━━━━━━━━━━━━━━━━━━━━━━━

💡 建議順序（完整行銷 Sprint）：
   1 → 5 → 2 → 3 → 4 → 8

💡 第一次來？輸入「我賣 [你的產品]」開始免費諮詢！

輸入數字或直接描述你的需求。
```

Based on user response, invoke the appropriate skill or command. If the user describes something in natural language, map it to the right workflow phase:

| User Says | Route To |
|-----------|----------|
| "我賣..." / "help me market" | `saleskit` (free consultation) |
| "成長策略" / "要做什麼" | `/mx-strategy` |
| "競品分析" / "competitor" | `market-intel` |
| "漏斗" / "funnel" / "旅程" | `plan-funnel-review` |
| "互動" / "私訊" / "FAQ" | `engage-operator` |
| "成交" / "為什麼不買" / "異議" | `conversion-closer` |
| "回購" / "老客" / "VIP" | `member-lifecycle` |
| "成效" / "KPI" / "回顧" | `growth-retro` |
| "品牌檢查" / "一致性" | `guard-brand` |
| "合規" / "療效" / "法規" | `brand-risk-review` |
| "上線" / "launch" | `campaign-ship` |
| "QA" / "測試旅程" | `journey-qa` |
| "做 LP" / "landing page" | `/mx-create` |
| "編輯" / "改 LP" | `/mx-edit` |
| "發文" / "社群" | `/mx-publish` |
| "影片" / "reels" | `/mx-reels` |
| "餘額" / "credits" | `/mx-status` |
