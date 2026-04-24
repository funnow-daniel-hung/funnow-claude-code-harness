---
name: storage-change-doc
description: Document ES, MongoDB, and MySQL schema changes with field specs, mapping, migration, and update strategy
---

## Storage Change Documentation

**When to use**: Any time Elasticsearch, MongoDB, or MySQL schemas are changed during implementation.

**Skip entirely** if no storage schemas were changed. Only include sections for storage types that were actually modified.

---

### Elasticsearch Changes

```markdown
### ES 變更

#### 新增/修改欄位

| Index | 欄位名稱 | Type | 說明 |
|-------|----------|------|------|
| sell_product | sell_time_hhmm | keyword | 商品可售時段 HH:mm (UTC) |

#### Mapping 變更

```json
PUT /sell_product/_mapping
{
  "properties": {
    "sell_time_hhmm": { "type": "keyword" }
  }
}
```

#### 舊資料更新方式
[說明策略：update_by_query / reindex / 新 index 等]
[說明原因：為何選此策略]

#### Query 範例
[若搜尋邏輯複雜，附上 ES query 範例片段]
```

---

### MongoDB Changes

```markdown
### MongoDB 變更

#### 新增/修改欄位

| Database | Collection | 欄位 | Type | 說明 |
|----------|-----------|------|------|------|
| myLogDB | smsLogs | newField | string | 說明 |

#### Index 變更

| Index Name | 說明 |
|-----------|------|
| IDX_{Collection}_{Field}_{Type} | [用途說明] |

命名規範依 CLAUDE.md：`IDX_{CollectionName}_{DocumentFieldName}_{IndexType}`
```

---

### MySQL Changes

```markdown
### MySQL 變更

#### 新增/修改欄位

| Table | Column | Type | 說明 |
|-------|--------|------|------|
| Orders | NewColumn | VARCHAR(50) | 說明 |

#### Index 變更

| Index Name | 說明 |
|-----------|------|
| IDX_{TableName}_{ColumnName} | [用途說明] |

#### Migration

- GOOSE Migration: `GOOSE-{N}` (去除 zero-padding)
- 檔案: `migrations/mysql/{migration_file}`
- 內容摘要: [簡述 migration 做了什麼]
```

---

### Rules

1. 繁體中文說明，技術名詞保留英文
2. 每個新增欄位都必須有 Type 和說明
3. Index 命名必須遵守 CLAUDE.md 規範
4. 舊資料更新策略必須說明選擇原因
5. 若有 migration 檔案，必須標註 GOOSE/MONGO 編號
6. 只包含有改動的儲存類型，不留空章節
