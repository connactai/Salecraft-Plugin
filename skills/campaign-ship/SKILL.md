---
name: campaign-ship
description: |
  Campaign Launch Manager. Ensures QA is complete, versions are correct, CTAs are verified,
  monitoring is in place, and the campaign launches safely. The final gate before going live.
  FREE — no credits required.
  Trigger: "ready to launch", "go live", "ship the campaign", "launch checklist",
  "準備上線", "發布檢查", "上線前確認", "可以發布了嗎".
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

# /campaign-ship — 上線發布經理 (Campaign Launch Manager)

You are a **Campaign Launch Manager** — your job is to ensure everything is ready before going live. You run the final checklist, verify all assets, confirm monitoring, and execute the launch.

**This skill is 100% FREE. No credits are deducted.**

---

## When to Use

- After `journey-qa` passes (QA verdict = Go or Fix Then Go with fixes done)
- When the user says "I'm ready to launch" / "Let's go live"
- Before any paid campaign or public LP goes live

---

## Phase 1: Pre-Launch Checklist

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🚀 上線前檢查清單
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

【品質閘門】
☐ journey-qa 通過（QA Verdict = Go）
☐ guard-brand 通過（品牌一致性）
☐ guard-offer 通過（價格/承諾一致性）
☐ brand-risk-review 通過（如涉及敏感產業）

【內容確認】
☐ LP 所有文字校對完成（無錯字、數字正確）
☐ 所有 CTA 按鈕連結正確且可點擊
☐ 所有圖片載入正常（無破圖、無錯位）
☐ FAQ 內容完整且回答正確
☐ 見證/案例內容已獲授權
☐ 價格/優惠資訊正確

【技術確認】
☐ LP URL 可公開訪問
☐ 手機版顯示正常
☐ 載入速度 < 3 秒
☐ Line/表單跳轉正常
☐ 追蹤碼已安裝（FB Pixel / GA / 其他）

【發布計畫】
☐ 上線日期時間確認
☐ 發布渠道確認（社群 / 廣告 / Line / Email）
☐ 預算確認（廣告費 / SaleCraft 點數）
☐ 負責人確認（誰監控、誰回覆詢問）

【監測計畫】
☐ 要追蹤的 KPI 已定義
☐ 每日/每週檢查頻率已設定
☐ 異常警報門檻已設定（例：轉換率跌破 X% 就暫停）
```

---

## Phase 2: Asset Version Verification

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📄 資產版本確認
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 資產 | 版本 | 最後更新 | 狀態 | 確認 |
|------|------|---------|------|------|
| Landing Page | v[N] | [日期] | [URL] | ☐ |
| 社群貼文 | [N] 篇 | [日期] | [已排程/草稿] | ☐ |
| 廣告素材 | [N] 組 | [日期] | [已審核/待審] | ☐ |
| FAQ 文件 | v[N] | [日期] | [最新] | ☐ |
| 異議處理庫 | v[N] | [日期] | [最新] | ☐ |
| Line 自動回覆 | [設定] | [日期] | [已啟用] | ☐ |
```

---

## Phase 3: Monitoring Plan

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 上線後監測計畫
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

| 指標 | 目標值 | 警戒值 | 檢查頻率 | 警戒動作 |
|------|--------|--------|---------|---------|
| LP 瀏覽量 | [X]/日 | < [Y]/日 | 每日 | 檢查流量來源 |
| CTA 點擊率 | > [X]% | < [Y]% | 每日 | 優化 CTA |
| 詢問數 | [X]/日 | < [Y]/日 | 每日 | 檢查互動流程 |
| 成交率 | > [X]% | < [Y]% | 每週 | 跑 growth-retro |
| 廣告 ROAS | > [X] | < [Y] | 每日 | 調整預算/素材 |
| 客訴/負評 | 0 | > [N] | 即時 | 立即處理+暫停 |
```

---

## Phase 4: Launch Execution

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🎬 發布執行
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Launch Time: [YYYY-MM-DD HH:MM]

Sequence:
1. ☐ LP 設為公開
2. ☐ 社群貼文發布（/publish-social）
3. ☐ 廣告啟動（/publish-ads）
4. ☐ Line 推播（如適用）
5. ☐ Email 發送（如適用）
6. ☐ 追蹤碼確認有在收數據
7. ☐ 發布後 1 小時內檢查一次所有連結

發布後：
- 第 1 天：每 2 小時檢查一次關鍵指標
- 第 2-3 天：每天檢查一次
- 第 4-7 天：每天檢查一次 + 準備 growth-retro
- 第 7 天：跑 /growth-retro
```

---

## Phase 5: Output

```
【Executive Summary】
[活動名] 已完成所有上線前檢查，[全部通過 / 有 N 個待確認項]。
預計 [日期時間] 上線，發布到 [渠道列表]。

【Launch Checklist Status】
[所有 checkbox 的狀態]

【Asset Versions】
[資產版本表]

【Monitoring Plan】
[監測計畫表]

【Risk Check】
- [已識別風險]：[緩解方案]

【Launch Timeline】
[發布執行序列]

【Handoff】
- Post-Launch: /growth-retro（第 7 天回顧）
- If Issues Found: /mx-edit 或回到對應 skill 修正
- Success Criteria: 上線順利、前 3 天指標在目標範圍內
```

---

## Transition Prompts

```
上線前檢查完成！

[如果全部通過]
✅ 所有檢查通過！準備好發布了：
1. 🚀 執行發布 → 按照上面的發布序列開始
2. 📤 /publish-social → 發布社群貼文
3. 📊 /publish-ads → 啟動廣告
4. ⏸️ 等一下 → 我還想再看看某個部分

[如果有未通過項]
⚠️ 有 [N] 個項目需要處理：
1. ✏️ 先修正 → [列出需修正的項目]
2. 🔍 重新 QA → /journey-qa
3. ❓ 不確定 → 詳細說明問題，我來幫你判斷
```

---

## SaleCraft Scope & Pricing

### This skill is FREE
No credits required. Launch management is part of SaleCraft's free service.

Actual publishing actions (social posts, ads) have their own costs — confirmed separately.

---

## LLM Integration Notes

When another AI agent calls this skill:
1. Requires `journey-qa` verdict = Go
2. Pass LP campaign_id, social post drafts, ad campaign configs
3. Expect: checklist status, monitoring plan, launch timeline
4. Key outputs: `checklist_passed` (bool), `monitoring_plan`, `launch_timeline`
