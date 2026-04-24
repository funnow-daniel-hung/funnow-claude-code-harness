# Plan to Jira

<context>
你現在扮演的角色是：資深後端工程師 + 技術 PM，正在幫工程師把一個需求轉成可執行的實作計畫。

**專案背景**（詳細參考 `$REPO_ROOT/CLAUDE.md`）：
- 從 CLAUDE.md 載入專案的技術棧、架構、命名慣例、錯誤處理等規範
- 任何實作計畫都必須符合這個 codebase 的結構與慣例，不是通用計畫

你的任務：透過結構化的三層確認，確保你完全理解需求後，才產出實作計畫。
</context>

## Input

$ARGUMENTS

## Instructions

> **Plan Mode optional**: For complex tasks, Plan Mode can improve reasoning quality. You may suggest it but do NOT block progress:
> `💡 Tip: Enable Plan Mode (Shift+Tab twice) for better results on complex plans. Continue without it if you prefer.`

Follow these steps in exact order:

### Step 1: Parse Input

Parse the user's input from `$ARGUMENTS`. Extract:

1. **Jira ticket key** (required): Must match the pattern `{{JIRA_PROJECT_KEY}}-\d+` (e.g., `{{JIRA_PROJECT_KEY}}-15801`). If no ticket key is found, ask the user to provide one. Do NOT proceed without a valid ticket key.
2. **Plan description** (required): The feature, bug, or task to plan. This is the remaining text after extracting the ticket key.

If either is missing, ask the user to provide both before proceeding.

Examples of valid input:
- `{{JIRA_PROJECT_KEY}}-15801 Refactor the order attendance flow to use clean architecture`
- `{{JIRA_PROJECT_KEY}}-16200 Fix race condition in concurrent booking`
- `{{JIRA_PROJECT_KEY}}-17000 Add OAuth login support for module X`

### Step 2: Three-Layer Confirmation (MANDATORY — do NOT skip)

這是強制步驟。每一層都必須等用戶明確說「ok」或「confirm」才能繼續下一層。

**Chain of Thought 思考順序**：先理解商業問題 → 再想技術方向 → 最後才看 code 細節。不能顛倒。

---

#### Layer 1：商業邏輯確認（≤300字）

用自己的話說清楚以下四點，**合計不超過 300 字**。如果你超過 300 字還沒說清楚，代表你自己還不清楚，必須先用 `AskUserQuestion` 問用戶，不能硬寫。

輸出格式：
```
**[Layer 1] 商業邏輯確認**

**問題**：[這個需求要解決什麼問題？現在哪裡痛？]
**影響對象**：[誰會受影響？使用者？內部系統？]
**成功標準**：[怎樣算做完？用戶能做到什麼事？]
**Scope 邊界**：[哪些在 scope？哪些不在？]

---
請確認以上理解是否正確，或補充我遺漏的部分。確認後輸入 ok 繼續。
```

**STOP HERE AND WAIT** for user to say ok before proceeding to Layer 2.

**好的 Layer 1 範例**：
> 問題：結帳流程不支援手機條碼載具，使用者無法在結帳時歸戶發票。
> 影響對象：台灣地區消費者（Web 商城、App、第三方合作站）。
> 成功標準：結帳頁可輸入手機條碼（/+7碼英數），訂單建立後正確傳給發票 API，訂單詳細頁顯示載具資訊。
> Scope 邊界：只做台灣地區、只做有過金流的方案，會員頁設定載具不在這個 sprint。

**壞的 Layer 1 範例**（不允許）：
> 「這個需求是關於發票載具的功能，需要在系統中加入手機條碼的支援，提升使用者體驗，並符合相關規範。」
> → 太模糊、沒有邊界、沒有成功標準，等於沒說。

---

#### Layer 1.5：Wiki + Graphify Context（自動，不需用戶確認）

Layer 1 確認後、進入 Layer 2 前，靜默載入相關背景知識：

1. **個人筆記 Wiki**：檢查 `~/wiki/index.md` 是否有與本次需求相關的頁面。如果有，讀取相關頁面。
2. **專案知識圖譜**：檢查 `graphify-out/GRAPH_REPORT.md` 和 `graphify-out/wiki/index.md` 是否存在。如果有，讀取與需求相關的 community 和 god node 資訊，了解相關程式碼的架構關聯。

將收集到的背景知識作為 Layer 2 技術思路的輸入。如果兩者都沒有相關內容，直接跳到 Layer 2。

這一步不需要輸出給用戶，靜默執行即可。

#### Layer 2：技術思路確認（≤500字）

Layer 1 確認後才能進行。用以下格式說清楚技術方向，**合計不超過 500 字**。超過 500 字代表思路不清晰，必須先精簡或問用戶。

**⚠️ Route 路徑強制查驗規則**：如果需求涉及任何 API endpoint（新增、修改、或引用既有），你**必須**用 Grep 查 codebase 中實際的 route 定義，**禁止憑需求描述猜測路徑**。具體做法：
1. `Grep` 搜尋相關關鍵字（例如功能名、資源名）在 `api/` 目錄下的 router/route 定義檔
2. 找到實際註冊的路徑後，在 Before/After 中引用 **Grep 查到的真實路徑**
3. 如果是新增 endpoint，查同功能模組的既有 route pattern 來決定新路徑的命名風格（例如 `/v1/movie-order` 而非 `/v1/order/movie`）
4. 在輸出中附上查到的 route 定義位置（檔案:行號）作為證據

輸出格式：
```
**[Layer 2] 技術思路確認**

**Before**：[現在的程式/流程是怎樣的？哪個檔案/函式/欄位負責這件事？]
**After**：[改完後程式/流程變成怎樣？新增/修改了什麼？]
**Route 證據**：[Grep 查到的實際 route 定義，含檔案:行號]
**做法**：[大概怎麼改？影響哪些 layer？用什麼 pattern？]
**不確定的部分**：[我還不清楚的地方，需要看 code 或你確認的]

---
請確認技術方向是否正確，或告訴我哪裡理解錯了。確認後輸入 ok 繼續。
```

**STOP HERE AND WAIT** for user to say ok before proceeding to Layer 3 / Step 3.

**好的 Layer 2 範例**：
> Before：訂單建立時 `forder/create.go` 呼叫發票 API，沒有傳入 CarrierNum，發票預設走會員載具（CarrierType=1）。DB 的 Orders 表沒有 invoice_carrier_num 欄位。
> After：結帳 API 接收 carrier_num 參數，存入 Orders 表新欄位，呼叫發票 API 時帶入 CarrierNum。
> 做法：新增 DB migration、修改 order domain entity、修改發票呼叫邏輯、修改結帳 API handler。
> 不確定：發票 SDK 是自己封裝的還是直接 HTTP call？carrier_num 驗證在哪一層做？

**壞的 Layer 2 範例**（不允許）：
> 「需要修改後端程式碼，新增欄位並串接綠界 API。」
> → 沒有 before/after、沒有具體檔案、沒有列出不確定的部分。

### Step 3: Invoke the Planner Agent

Dispatch a planner agent to read the codebase and generate a structured implementation plan.

Invoke `Agent` with `subagent_type: "planner"`, `model: "opus"`:

**主 agent 必須在 prompt 中附上 Layer 2 過程中查到的所有 code context**（檔案路徑、行號、code snippet）。planner 不應該自己重新探索 codebase — 所有需要的資訊都由主 agent 提供。

**Prompt template** (static sections first for KV cache efficiency):
```
## Instructions
1. Read CLAUDE.md at $REPO_ROOT/CLAUDE.md for project conventions
2. DO NOT explore the codebase yourself — all relevant code context is provided below in "Codebase Context"
3. If the provided context is insufficient for a specific detail, you may use Grep/Glob to verify that ONE detail, but do NOT broadly explore
4. **Route path verification (MANDATORY)**: For every API endpoint mentioned in the plan, you MUST Grep the actual route registration in the codebase to confirm the path is correct. NEVER copy a route path from the context without verifying it. If a route doesn't exist yet, Grep the same module's existing routes to match the naming convention.
5. Return a step-by-step plan with: exact file paths, phases, risks, dependencies
6. If anything is unclear, STOP and list your questions — do NOT assume

## Output Format
Return a structured plan with phases, each containing: files to change, what to change, why, risks.
Follow the Plan Format defined in your agent definition.

## Task
Plan implementation for: {plan description from Layer 2}

## Context
- Jira ticket: {ticket key}
- Business context: {Layer 1 summary}
- Technical direction: {Layer 2 summary}

## Codebase Context (collected by main agent — DO NOT re-explore)
{paste all file paths, line numbers, and code snippets discovered during Layer 2}
```

Planner 產出詳細版後，**先不要把整份詳細計畫貼給用戶**，直接進入 Step 3.2 把詳細版濃縮成摘要版給用戶看。

### Step 3.2: 摘要版對話確認（MANDATORY）

Planner 產出詳細計畫後，主 agent **從詳細計畫濃縮出摘要版**給用戶，讓用戶先確認思路，**不要把完整詳細版直接貼給用戶**。摘要版跟詳細版必須一致（同源濃縮），不能多、不能少。

**分組邏輯**（擇一）：
- **有影響 API endpoint**：按 endpoint 分組（`POST /v1/order`、`PATCH /v1/pub/order/{orderid}` ...）
- **無 endpoint 改動**（例如純 module 重構、schedule job、內部 service）：按外→內層級分組（`api/` handler → `api/services/` → `api/flow/` → `module/usecase/` → `module/repository/` → `lib/`），每一層底下列出涉及的檔案

**每個檔案一行**，用「檔名 — 動詞 + 改動重點（含用到新/舊結構的提示）」的句型，**不要放 code snippet**。重點讓用戶一眼看懂：
- 新增了什麼（新欄位、新函式、新 struct）
- 用到舊的還是新的（例如「沿用既有 X」「新增 Y 取代 Z」「移除 W」）
- 跨層依賴（例如「tx 外 log error」「失敗 rollback」「AfterHook 處理」）

**輸出格式**：

```
**[Step 3.2] 計畫摘要（濃縮版）**

POST /v1/order
- req_valid.go — Request 加 xxx 欄位 + Bind() 加 Validate
- params.go — UserInput / UserParams 傳遞 xxx
- create_order_service.go — create() 呼叫新的 insertXxx 寫 DB
- create_order.go — 移除 updateEmail，新增 upsertMemberXxx（tx 外 log error）

POST /v1/movie-order
- create_movie.go — 同 /v1/order 模式

...（其他 endpoint 或層級）

---

這是思路摘要。詳細版（含 code snippet）會在你確認 ok 後產出貼到 Jira。
- 思路對 → 輸入 `ok`
- 要調整 → 直接說哪裡要改（會 replan）
```

**STOP HERE AND WAIT** for user response.

**User 回應處理**：
- 說 `ok`/`confirm`/沒問題 → 進入 Step 3.5
- 提出任何修改（例如「POST /v1/order 不要移除 updateEmail」「改用 domain event 而不是 AfterHook」）→ **必須 replan**：帶著用戶的修改意見回到 Step 3 重新呼叫 planner agent，產出新的詳細計畫，再回到 Step 3.2 產生新的摘要，再次等用戶確認。跟 Plan Mode 一樣，直到用戶 ok 為止。

### Step 3.5: Plan Critique (唱反調)

After the planner produces the plan and the user has no more edits, dispatch a **critique agent** before finalizing. This agent's sole job is to find problems — it is NOT allowed to confirm the plan is good.

Invoke `Agent` with `model: "sonnet"`:

**Prompt template**:
```
## Role
你是計畫批判者。你的任務不是確認計畫能不能用，而是找出問題。

## Plan to Critique
{paste the plan produced by Step 3}

## Instructions
1. Read CLAUDE.md at $REPO_ROOT/CLAUDE.md for project conventions
2. For every file path mentioned in the plan, verify it exists using Glob/Grep
3. Check for these common planning mistakes:
   - **Route 路徑錯誤**（最常出錯）：Grep 每個計畫中提到的 API route，確認路徑跟 codebase 實際註冊的一致。如果是新 route，確認命名風格跟同模組既有 route 一致
   - 漏改的檔案（例如改了 domain interface 但沒列 mock regeneration）
   - 改動順序有依賴問題（例如先改 handler 但 service method 還不存在）
   - 過度設計（能用 3 行解決的事情用了一整套 pattern）
   - 遺漏的 error handling 或 edge case
   - 與 CLAUDE.md 慣例衝突的寫法
4. DO NOT modify any files — analysis only

## Anti-laziness Rules
- 「看起來沒問題」不是有效結論 — 你必須逐項檢查後才能說沒問題
- 「太花時間了」不是你該擔心的事
- 每個檢查點都要附上你實際驗證的證據（例如 Glob 結果、Grep 結果）

## Output Format
Return a list of issues found, each with:
- **Issue**: 問題描述
- **Severity**: CRITICAL / HIGH / MEDIUM / LOW
- **Evidence**: 你怎麼發現的（Glob/Grep 結果）
- **Suggestion**: 建議怎麼修

If genuinely no issues found after thorough checking, return "No issues found" with evidence of checks performed.
```

**After critique returns:**
- If CRITICAL or HIGH issues found → present to user and ask: 「批判者發現以下問題，要修正計畫再繼續嗎？」Then loop back to Step 3 to revise.
- If only MEDIUM/LOW or no issues → present findings as FYI, then proceed to Step 4.

### Step 4: Wait for Plan Confirmation

After the planner skill completes and the user has confirmed the plan, ask:

```
---

**The plan is ready to be written to Jira.**

- **Ticket**: {{JIRA_PROJECT_KEY}}-XXXXX
- **Action**: Add implementation plan as comment

Type **yes** to write this plan to Jira, or **no** to cancel.
```

**STOP HERE AND WAIT.** Do not proceed until the user responds.

- If the user says **yes** (or y, confirm, go, do it, ship it, send it): proceed to Step 5.
- If the user says **no** (or cancel, stop, abort): respond with "Plan cancelled. No changes were made to Jira." and STOP.

### Step 5: Format the Plan for Jira

Take the final confirmed plan and format it for Jira using the structure below. The plan has two audiences:
- **上半部（人看的）**：給 PM / 其他工程師快速理解這張卡在做什麼
- **下半部（agent 執行的）**：給實作 agent 用，必須詳細到能直接照做，不能有歧義

#### 輸出結構（依序）

計畫分兩個區塊，必須用以下分隔線明確標注：

```markdown
---
> **以上為人工確認區（PM / 工程師閱讀）。以下為 Agent 實作規格，確認上方內容後交由 Agent 執行。**
---
```

- **上半部（人看的）**：概述、需求、期望 Case 表格、API 變更文件、已完成改動 — 放在分隔線**之前**
- **下半部（agent 執行的）**：待實作、Testing Strategy、完成標準 — 放在分隔線**之後**

---

**1. 概述**（人看的，≤100字）

一段話說清楚這張卡的目的和背景。

---

**2. API 變更文件**（前端看這裡，有影響任何 endpoint 的 request/response 才寫，否則完全跳過）

「有影響 endpoint」的定義：新增 endpoint、廢棄 endpoint、修改現有 endpoint 的 request 參數、**或修改現有 endpoint 的 response 欄位（新增/移除/改型別）**——只要前端需要知道的都算。

**必須包含以下所有小節**（依序）：

**2a. Affected Endpoints 表格** — method, path, 說明（新增/修改/廢棄）

**2b. 新增/修改欄位表格** — 欄位名稱, 必填, 類型, 說明（enum 值用 backtick）

**2c. Response 說明** — response 有變更就列出變更的欄位；response 不變就寫一句「Response 不變」

**2d. Request 範例**（**必寫**，不是可選）— 用具體 JSON 範例呈現每種使用情境，讓前端一看就懂怎麼帶參數。每種主要情境各一個範例，包含：
- 新功能的各種組合（例如不同 type、有帶/沒帶可選欄位）
- 向下兼容的情境（舊 client 不帶新欄位）
- 與既有欄位互斥的情境（如果有）

**2e. Error Response 範例**（**必寫**，只要有新增或變更 error code）— 列出：
- 每個 error 的 JSON response 範例（含 `code` 和 `message`）
- 對應的 i18n key
- 新增/已存在 標注

**2f. Error Code 總覽表格** — code, i18n key, 說明, 狀態（新增/已存在）

Rules:
- 繁體中文說明，技術名詞保留英文
- 若多個端點共用邏輯，備註說明，不重複
- Request 範例的 JSON 要完整可讀，但只需要包含跟本次變更相關的重點欄位 + 最基本的必填欄位

---

**3. 需求與驗證規則**（後端看這裡）

**3a. 業務規則** — 條列式列出功能需求，包含：
- 哪些 endpoint / 功能受影響
- 每個欄位/行為的來源與邏輯
- 邊界條件（null 處理、空字串、無資料等）

**3b. 驗證流程**（如果有驗證邏輯）— 用 ASCII 流程圖或條列式呈現驗證步驟，讓人一眼看懂成功/失敗/降級的分支：

```
本地驗證 → 外部 API
            ├─ 成功 → 放行
            ├─ 失敗 → 擋住回錯
            └─ 超時 → 降級放行
```

**3c. 外部 API 回傳值對照表**（如果有打外部 API）— 用表格列出每個 return code 的處理方式，不要藏在 code snippet 裡：

```markdown
| RtnCode | 說明 | 處理 |
| --- | --- | --- |
| 1 | 成功 | 放行 |
| 9000001 | API 異常 | 降級放行，log warning |
```

**如果沒有驗證邏輯或外部 API，3b 和 3c 跳過。**

---

**4. 期望 Case 表格**（人看的，如果有多情境）

如果需求涉及多種情境（例如不同載具類型、不同地區、不同狀態），用表格列出所有 case：

```markdown
| 情境 | 欄位A | 欄位B | 欄位C |
| --- | --- | --- | --- |
| 情境描述 | 值 | 值 | 值 |
```

**如果情境單純（只有一種）則跳過此區塊。**

---

**5. 已完成的改動**（人看的 + agent 參考）

列出這張卡上已經改好的檔案，方便 agent 了解依賴現況、避免重工：

```markdown
| 檔案 | 改動 |
| --- | --- |
| `path/to/file.go` | 具體改了什麼 |
```

**如果沒有已完成改動則跳過此區塊。**

---

**6. 待實作**（給 agent 執行，必須詳細）

每個待實作項目格式如下：

```markdown
#### N. `path/to/file.go` — 方法名稱或改動描述

[一句話說明為什麼要改這裡，或現在缺什麼]

```go
// 具體的 code snippet，包含完整函式簽名、邏輯、error handling
// 如果是修改現有程式，標註要加在哪個位置（例如：在 return xxx 前新增）
```
```

規則：
- **每個項目都要有 code snippet**，不能只說「新增方法」或「修改邏輯」
- 如果是修改現有函式，要說明插入位置（在哪行前/後）
- 如果有 import 要新增，一併列出
- test 的 mock 如果需要更新，也要列出（包含 SQL pattern）

---

**7. Testing Strategy**（給 agent 執行）

條列要補的 unit test cases（build tag: `//go:build unit`）：
- 每個新方法的 happy path + error cases
- 需要更新的既有 test mock

---

**8. 完成標準**（checklist）

```markdown
- [ ] 具體可驗證的條件
```

---

Add a footer: `---\n*Plan generated by Claude Code on [today's date]*`

### Step 6: Write Plan to Jira

First, call `mcp__claude_ai_Atlassian__getJiraIssue` to verify the issue exists:
- `cloudId`: `{{JIRA_CLOUD_ID}}`
- `issueIdOrKey`: The extracted ticket key (e.g., `{{JIRA_PROJECT_KEY}}-15801`)

If the issue does NOT exist, inform the user and STOP.

If the issue exists, call `mcp__claude_ai_Atlassian__addCommentToJiraIssue` with:
- `cloudId`: `{{JIRA_CLOUD_ID}}`
- `issueIdOrKey`: The ticket key
- `commentBody`: The full formatted plan from Step 4, prefixed with `## Implementation Plan\n\n`
- `contentFormat`: `markdown`

### Step 7: Confirm Success

After the Jira API call succeeds, display:

```
Plan successfully written to Jira.

- **Issue**: [{{JIRA_PROJECT_KEY}}-XXXXX](https://{{JIRA_DOMAIN}}/browse/{{JIRA_PROJECT_KEY}}-XXXXX)
- **Action**: Added implementation plan as comment
- **Title**: [issue summary]

The plan is now ready for an implementing agent to pick up.
```

If the Jira API call fails, display the error and offer to:
1. **Retry** — try again
2. **Copy** — display the plan for manual paste
3. **Cancel** — abort

## Critical Constraints

- **NEVER** write, modify, or execute any code files during this workflow
- **ALWAYS** use the Agent tool with `subagent_type: "planner"` for planning — do NOT plan manually
- **ALWAYS** wait for explicit user confirmation before any Jira API call
- **ALWAYS** use `markdown` as the contentFormat for Jira
- **ALWAYS** verify the issue exists before writing to it

## Critical Rules（最重要，重申一次）

1. **資訊不足就停，絕不假設**：任何不清楚的地方，用 `AskUserQuestion` 問，不要自己腦補填空
2. **Layer 1 超過 300 字 = 你不清楚**：必須先精簡或問用戶，不能硬寫超字數
3. **Layer 2 超過 500 字 = 思路不清晰**：必須先整理或問用戶
4. **每層都要 STOP 等 ok**：不能連續輸出 Layer 1 + Layer 2，必須逐層確認
5. **Before/After 是必填**：Layer 2 沒有具體 before/after 就是不合格，重寫
6. **不確定的部分必須明確列出**：不能用「其他細節待確認」帶過，要具體說是什麼不確定
7. **Step 3.2 摘要版不能跳過**：planner 產出詳細計畫後，不要把完整詳細版貼給用戶，一律先濃縮成摘要版讓用戶確認思路，ok 後才進 Step 3.5。用戶提出修改 → 一定要 replan（回到 Step 3），不能自己改摘要。
