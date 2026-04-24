---
name: integration-test
description: Run integration tests via Hurl against local API, auto-manage air server, generate .hurl files by domain, show detailed results
---

## Integration Test Skill (Hurl)

**When to use**: After implementation is complete and build passes. Tests the actual HTTP API against a running local server using [hurl.dev](https://hurl.dev/).

**Prerequisite**: `brew install hurl` (if not installed, prompt user to install first)

---

### Step 0: Server Management (auto)

Automatically detect and manage the local server:

```bash
# 1. Probe if server is already running
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 3 http://localhost:8080/ 2>/dev/null || echo "000")

if [ "$HTTP_CODE" != "000" ]; then
  STARTED_BY_US=false
else
  cd $REPO_ROOT
  air &
  AIR_PID=$!
  STARTED_BY_US=true

  for i in $(seq 1 60); do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time 2 http://localhost:8080/ 2>/dev/null || echo "000")
    if [ "$HTTP_CODE" != "000" ]; then break; fi
    sleep 1
  done

  if [ "$HTTP_CODE" = "000" ]; then
    echo "ERROR: Server failed to start after 60s"
    kill $AIR_PID 2>/dev/null
    # STOP and report to user
  fi
fi
```

### Step 1: Determine Test Strategy

Based on changes, pick ONE:

- **New API endpoint** → Full test: happy path + bad request + auth + edge cases
- **Modified existing API** → Regression: replay known good cases + verify new behavior

### Step 2: Identify Domain & Endpoints

1. Read changed files to identify which domain(s) are affected
2. Map to domain directories under `tests/hurl/`
3. List all affected endpoints with their auth requirements

Domain mapping (based on service packages):
- `api/services/member/` → `tests/hurl/member/`
- `api/services/order/` or `api/services/sorder/` → `tests/hurl/order/`
- `api/services/branch/` → `tests/hurl/branch/`
- `api/services/search/` → `tests/hurl/search/`
- `api/services/review/` → `tests/hurl/review/`
- `api/services/product/` → `tests/hurl/product/`
- `api/services/payment/` or `api/services/txn/` → `tests/hurl/payment/`
- Other → use the service package name as domain directory

Auth detection from route files:
- **Public (pub)**: routes under `pub` group → no Cookie needed
- **Auth required**: routes with auth middleware → add `Cookie: {{cookie}}`
- **Admin**: routes with admin middleware → add admin auth header

### Step 3: Design Test Cases

For each endpoint, design test cases in a table before writing .hurl files.

#### New API:

| # | Case | Method | Path | Expected Status | Expected Assertions |
|---|------|--------|------|-----------------|---------------------|
| 1 | Happy Path | GET | /v2/... | 200 | code == 0, data exists |
| 2 | Bad Param | GET | /v2/...?bad=x | 400 | code == 10 |
| 3 | Missing Auth | GET | /v2/... (no cookie) | 401 | — |
| 4 | Not Found | GET | /v2/.../999999 | 404 | code == 16 |

#### Modified API (regression):

| # | Case | Method | Path | Expected Status | Verify |
|---|------|--------|------|-----------------|--------|
| 1 | Original Happy Path | GET | /v2/... | 200 | response structure unchanged |
| 2 | New Field Present | GET | /v2/... | 200 | new field exists + correct type |
| 3 | Old Field Intact | GET | /v2/... | 200 | existing fields still present |

### Step 4: Generate .hurl Files

Write `.hurl` files to `tests/hurl/{domain}/` directory.

**File naming**: `{action}_{resource}.hurl` (e.g., `get_profile.hurl`, `create_order.hurl`)

**Hurl file structure**:

```hurl
# tests/hurl/member/get_profile.hurl
# Domain: member
# Test: Get member profile

# --- Happy Path ---
GET {{host}}/v2/member/profile
Accept-Language: zh-TW
Cookie: {{cookie}}

HTTP 200
[Asserts]
jsonpath "$.code" == 0
jsonpath "$.data" exists
jsonpath "$.data.MemberID" exists


# --- Missing Auth ---
GET {{host}}/v2/member/profile
Accept-Language: zh-TW

HTTP 401
[Asserts]
jsonpath "$.code" == 20
```

**Variable conventions**:
- `{{host}}` — default `http://localhost:8080`
- `{{cookie}}` — session cookie for auth endpoints
- `{{api_version}}` — default `v2`

**Variables file** (`tests/hurl/vars.env`):
```
host=http://localhost:8080
cookie=YOUR_TEST_COOKIE_HERE
api_version=v2
```

**CRITICAL**: NEVER hardcode `localhost:8080` in .hurl files. Always use `{{host}}`.
**Reason**: Allows reuse against dev/stg environments by changing vars.env.

### Step 5: Execute Tests

```bash
# Run all tests for affected domain
hurl --test \
  --variables-file tests/hurl/vars.env \
  --report-junit tests/hurl/report.xml \
  tests/hurl/{domain}/*.hurl

# Run a specific test file
hurl --test \
  --variables-file tests/hurl/vars.env \
  tests/hurl/{domain}/{test_file}.hurl

# Debug a failing test
hurl --very-verbose \
  --variables-file tests/hurl/vars.env \
  tests/hurl/{domain}/{test_file}.hurl
```

### Step 6: Present Results (2 layers)

#### Layer 1: Summary Table (for Jira + terminal)

```markdown
### 整合測試結果

- 測試環境: localhost:8080 (air)
- 工具: hurl
- 策略: [新增 API 測試 / Regression 測試]
- 結果: X/Y passed

| # | Case | Endpoint | Status | Result |
|---|------|----------|--------|--------|
| 1 | Happy Path | GET /v2/member/profile | 200 | PASS |
| 2 | Bad Param | GET /v2/member/profile?bad=x | 400 | PASS |

> Hurl files: `tests/hurl/{domain}/`
> JUnit report: `tests/hurl/report.xml`
```

#### Layer 2: Detail (terminal only, NOT Jira)

For failed tests: show `hurl --very-verbose` output.
For passing tests: brief one-line summary.

### Step 7: Jira Summary Block

For Phase 4 of /ship, output ONLY Layer 1 (same as above, without Layer 2).

### Step 8: Cleanup

```bash
if [ "$STARTED_BY_US" = "true" ]; then
  kill $AIR_PID 2>/dev/null
  wait $AIR_PID 2>/dev/null
fi
```

After cleanup, remind user:
- If POST/PUT tests created data, list what was created and ask about cleanup
- Report hurl file paths for future regression runs
- Remind: `hurl --test --variables-file tests/hurl/vars.env tests/hurl/{domain}/*.hurl`

### Rules

1. 繁體中文說明，技術名詞保留英文
2. FAIL 時立即停下報告，不要自動修
3. POST/PUT 新增的測試資料，結束後提醒用戶清理
4. .hurl 檔案用 `{{host}}` 變數，NEVER hardcode localhost。Reason: 讓 .hurl 可跨環境使用
5. 一個 .hurl 檔案 = 一個 resource 的所有 test cases（happy + error）
6. 按 domain 分目錄，目錄名對應 service package
7. `vars.env` 不 commit（加入 .gitignore），`.hurl` 檔案要 commit
8. Server 管理：偵測 → 沒跑就開 → 測完如果是自己開的就關
9. 已存在的 .hurl 檔案不要覆蓋，除非 API 有改動 — 追加新 test case 或修改受影響的斷言
10. 如實回報：測試沒跑過就說沒跑過，跑過 PASS 也不要加免責聲明說「可能還有問題」
