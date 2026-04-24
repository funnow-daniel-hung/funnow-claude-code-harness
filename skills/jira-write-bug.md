---
name: jira-write-bug
description: Format implementation summary for Bug tickets following Jira conventions
---

## Bug Ticket Jira Summary Format

When writing a summary for a **Bug** type Jira ticket, use this exact format.

### Title Format

`{component} - {修改內容}`

Example: `Review API - 修正產品評論數量與列表不一致`

### Content Format

```markdown
## {component} - {修改內容}

### Precondition
[測試前的前置條件（帳號狀態、測試環境設定、資料狀態）]

### Reproduce Step
1. [步驟一]
2. [步驟二]
3. ...

### Actual Result
[實際發生的結果]
[附上相關 API payload/response、Log 連結、截圖]

### Expect Result
[期望應該出現的正確結果]

### Root Cause
[根本原因分析；若未確認填「TBD」]
[說明技術層面的原因，例如 SQL JOIN 邏輯不一致、欄位類型錯誤等]

### Solution
[實際修復方式]
[例如：「修正 API 回傳參數」、「統一 JOIN 邏輯為 BID-level」、「增加錯誤處理」]
```

### Rules

1. **繁體中文為主**，技術名詞保留英文原文
2. **精簡去冗餘**，去除重複、口語化內容，保留核心資訊
3. **保留技術細節**：DB 欄位名、config key、URL、API path 必須完整保留
4. **不要自行編造內容**，僅從實際實作和 Jira 現有資訊中提取
5. 此格式後面會接上其他 skill 產生的區塊（API 變更、ES/DB 變更、測試注意事項等），不要重複這些內容
