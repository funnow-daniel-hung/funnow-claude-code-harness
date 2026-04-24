---
name: api-change-doc
description: Generate API change documentation with endpoint tables, field specs, and request/response examples
---

## API Change Documentation

**When to use**: Any time API endpoints are added, modified, or deprecated during implementation.

**Skip entirely** if no API endpoints were changed.

### Format

#### 1. Affected Endpoints Table

```markdown
### API 變更

#### Endpoints

| Method | Endpoint | 說明 |
|--------|----------|------|
| GET | /v2/pub/branch | [新增/修改] - [一句話說明] |
| GET | /v2/branch | [新增/修改] - [一句話說明] |
```

**備註**: [補充端點間的關係，例如「pub 與非 pub 版本為同一 handler，差別僅在是否需要登入」]

#### 2. Field Changes Table

```markdown
#### 新增/修改欄位

| 欄位名稱 | 必填 | 類型 | 說明 |
|----------|------|------|------|
| filter_times[] | 否 | string array | 時段篩選，格式 `"hh:mm"`，可多選 |
```

**規則**：說明欄中的 enum 值、欄位名稱、格式字串一律用 backtick 包起來。例如：
- ✅ 來源，如 `internal`、`google_maps`
- ✅ 摘要類型，如 `overview`、`product_usp`、`vibe_service`
- ❌ 來源，如 internal、google_maps（未包 backtick）

#### 3. Request & Response Examples

根據 handler 實際使用的 `apires` 型別選擇對應範例格式（讀 handler 原始碼確認）：

---

##### apires.Data（單筆資料，最常用）

Handler: `c.JSON(http.StatusOK, apires.Data{Base: apires.DefOKBase, Data: ...})`

**Happy Path (200 OK)**:
```
GET /v2/pub/branch/:bid/review-summary
Accept-Language: zh-TW
```

**Response**:
```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "field1": "value",
    "field2": 123
  }
}
```

---

##### apires.List（列表＋總筆數，data 與 total_count 同層）

Handler: `c.JSON(http.StatusOK, apires.List{Base: apires.DefOKBase, Data: ..., Count: n})`

**Happy Path (200 OK)**:
```
GET /v2/pub/branch?filter_regionid=1&starttime=2026-03-02&size=10
```

**Response**:
```json
{
  "code": 0,
  "message": "OK",
  "total_count": 42,
  "data": [...]
}
```

---

##### apires.ListV2（列表＋總筆數，data.list 包裝）

Handler: `c.JSON(http.StatusOK, apires.ListV2{Base: apires.DefOKBase, Data: apires.ListData{...}})`

**Response**:
```json
{
  "code": 0,
  "message": "OK",
  "data": {
    "total_count": 42,
    "list": [...]
  }
}
```

---

##### apires.TimezoneData（單筆＋時區）

Handler: `c.JSON(http.StatusOK, apires.TimezoneData{Base: apires.DefOKBase, Timezone: ..., Data: ...})`

**Response**:
```json
{
  "code": 0,
  "message": "OK",
  "timezone": "Asia/Taipei",
  "data": {...}
}
```

---

##### apires.Base（無 data，僅狀態，用於 create/update/delete）

Handler: `c.JSON(http.StatusOK, apires.DefOKBase)`

**Response**:
```json
{
  "code": 0,
  "message": "OK"
}
```

---

##### Error Responses（統一格式，由 errors.Throw / errors.Error 產生）

> **注意**：這個 codebase **不使用** `"error": null` 或 `"success": true`。所有 error 都是 `code` + `message`。

**400 Bad Request — 參數錯誤** (`CODE_INVALID_PARAMS = 10`):
```json
{
  "code": 10,
  "message": "params_err"
}
```

**401 Unauthorized — 未登入或 token 無效**:
```json
{
  "code": 20,
  "message": "unauthorized"
}
```

**404 Not Found — 資料不存在** (`CODE_NOT_EXISTS = 16`):
```json
{
  "code": 16,
  "message": "data_not_found"
}
```

**500 Internal Server Error** (`CODE_UNKNOWN_ERR = -1`):
```json
{
  "code": -1,
  "message": "unknown_error"
}
```

**400 多欄位驗證錯誤 — apires.Errs**（有多個 field 錯誤時）:
```json
{
  "code": 10,
  "message": "params_err",
  "errs": {
    "email": ["invalid format"],
    "phone": ["required"]
  }
}
```

#### 4. Response Changes (if any field added/removed/modified in response body)

```markdown
#### Response 變更

| 欄位 | Before | After | 說明 |
|------|--------|-------|------|
| xxx  | -      | string | 新增欄位 |
```

### Rules

1. 繁體中文說明，技術名詞保留英文
2. 每個新增/修改欄位都必須有類型和說明
3. Request 範例必須包含 Happy Path 和 Bad Request 各至少一個
4. 若多個端點共用相同邏輯，在備註說明，不需重複寫範例
5. 若端點有互斥參數，必須在說明中標註（例如「filter_times[] 與 sell_time_range_type 互斥」）
