---
name: jira-write-task
description: Format implementation summary for Task tickets following Jira conventions
---

## Task Ticket Jira Summary Format

When writing a summary for a **Task** type Jira ticket, use this exact format.
All sections are optional — only include sections that have relevant content.

### Title Format

`{component} - {修改內容}`

Example: `Search API - 新增 filter_times 時段篩選功能`

### Content Format

```markdown
本票為「{父票 Summary}」({父票 Key}) 的子任務。
目的是{一句話說明此票要達成的事}。

## 目標
- [目標一]
- [目標二]

## 邏輯規則
- [條件分流邏輯，例如 switch/case 的業務分支]
- [預設值說明]
- [覆寫規則]

### 期望 Case 表格
[若有多種情境，用表格列出每種情境的參數值和預期行為]

| 情境 | 參數A | 參數B | 行為 |
|---|---|---|---|
| 情境一 | 值 | 值 | 預期行為 |
| 情境二 | 值 | 值 | 預期行為 |

## 實作
[按架構層級分組，每組用表格列出檔案和改動]

### DB / Migration
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### Entity
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### Config / Client
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### Service
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### API
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### Testing
| 檔案 | 改動 |
|---|---|
| `path/to/file` | 改動說明 |

### 注意事項
- [影響範圍、breaking change、靜默行為等需要特別注意的事項]

### {領域特有 Section}
[若有領域特有的資訊需要記錄，例如第三方 API 版本狀態、外部系統整合狀態等，自行新增 section]

## 驗收標準
- [x] [已完成的驗收條件]
- [ ] [待完成的驗收條件]
```

### Section 說明

| Section | 何時使用 |
|---|---|
| 目標 | 必填，列出此票要達成的具體目標 |
| 邏輯規則 | 有業務分流邏輯、條件判斷時使用 |
| 期望 Case 表格 | 有多種情境且每種情境參數不同時使用 |
| 實作 | 必填，按架構層級分組（DB → Entity → Config → Service → API → Testing），用表格列出每個檔案的改動 |
| 注意事項 | 有 breaking change、靜默行為、影響範圍等需要特別注意時使用 |
| 領域特有 Section | 視需求自行新增，例如「ECPay API 版本狀態」「第三方整合狀態」等 |
| 驗收標準 | 必填，完成的打勾、未完成的留空 |

### 實作分組規則

按架構層級從底層到上層排列，跳過沒有改動的層級：

1. **DB / Migration** — schema 變更、migration 檔案
2. **Entity** — `edb`、`emongo`、`eredis` 的 struct 定義
3. **Config / Client** — config 新增、第三方 client 改動
4. **Service** — `api/services/` 的業務邏輯
5. **API** — handler、route、middleware
6. **Testing** — unit test、integration test

每組用 `| 檔案 | 改動 |` 表格格式，不要用 bullet list。

### Rules

1. **繁體中文為主**，技術名詞保留英文原文
2. **精簡去冗餘**，去除重複、口語化內容，保留核心資訊
3. **保留技術細節**：DB 欄位名、config key、URL、API path 必須完整保留
4. **無相關內容的章節直接省略**，不留空章節
5. **不要自行編造內容**，僅從實際實作和 Jira 現有資訊中提取
6. **實作用表格不用 bullet**，每個架構層級一張表
7. **若有現有 description**，讀取後保留人工確認區上方的結構，將實作結果整合進去，而不是另起新格式
8. 此格式後面會接上其他 skill 產生的區塊（API 變更、ES/DB 變更、測試注意事項等），不要重複這些內容
