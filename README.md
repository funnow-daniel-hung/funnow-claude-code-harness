# Claude Code Harness

一套基於 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 的 **Harness Engineering** 實戰架構,透過 slash commands、skills、subagent 系統,實現結構化規劃、多 Agent 協作審查、Jira 驅動的端到端開發工作流。

> **注意**:這是架構參考,不是可直接安裝的套件。原本為 Go 微服務專案設計(Gin/GORM/MongoDB),內容已去識別化並以占位符標記專案專屬設定,請作為藍圖參考依自己的專案調整使用。

---

## 快速開始:專案占位符

本 repo 所有檔案中的以下占位符需要替換成你自己專案的值:

| 占位符 | 說明 | 範例值 |
|--------|------|--------|
| `$REPO_ROOT` | 主 repo 根目錄(絕對路徑) | `/Users/you/code/myapp` |
| `$WORKTREE_ROOT` | git worktree 存放根目錄 | `/Users/you/code/myapp-worktrees` |
| `{{PROJECT}}` | 專案名稱 | `myapp` |
| `{{JIRA_DOMAIN}}` | Jira 子網域 | `yourcompany.atlassian.net` |
| `{{JIRA_CLOUD_ID}}` | Atlassian cloudId(由 `mcp__claude_ai_Atlassian__getAccessibleAtlassianResources` 取得) | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| `{{JIRA_PROJECT_KEY}}` | Jira project key | `PROJ` |
| `{{GO_MODULE}}` | Go module 路徑 | `github.com/you/myapp` |
| `{{API_GUIDE_URL}}` | 團隊 API guide 連結(若無可移除) | `https://wiki.yourcompany.com/api-guide` |

**建議做法**:fork 本 repo 後全檔搜尋取代一次就好。

---

## 目錄

- [整體架構](#整體架構)
- [知識系統](#知識系統)
- [Commands 完整清單](#commands-完整清單)
- [兩個核心命令](#兩個核心命令)
- [兩個命令怎麼串接](#兩個命令怎麼串接)
- [完整流程圖](#完整流程圖)
- [目錄結構](#目錄結構)
- [Skills、Rules、Agents 參考](#skills-參考)
- [Agent 定義檔設計說明](#agent-定義檔設計說明)
- [使用的技術](#使用的技術)

---

## 整體架構

```
┌─────────────────────────────────────────────────────────────────┐
│                     Claude Code Harness                         │
│                                                                 │
│  ┌─ 你直接用的 ────────────────────────────────────────────┐    │
│  │  /plan-to-jira    規劃需求 → 寫 Jira                    │    │
│  │  /ship            端到端:規劃→實作→審查→測試→PR         │    │
│  │  /wiki            整理個人筆記                           │    │
│  │  /graphify        掃描專案建立知識圖譜                   │    │
│  └──────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ▼                                                        │
│  ┌─ 背後自動派出的 Agent ──────────────────────────────────┐    │
│  │  planner (opus)          讀 codebase 產計畫              │    │
│  │  go-reviewer (sonnet)    程式碼審查                      │    │
│  │  go-build-resolver       修 build 錯誤                   │    │
│  │  go-security-reviewer    安全漏洞檢查                    │    │
│  │  go-tdd-guide (sonnet)   寫 unit test                   │    │
│  │  critique agent          唱反調質疑計畫                  │    │
│  │  4x haiku agents         並行產文件                      │    │
│  └──────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ▼                                                        │
│  ┌─ 知識來源 ──────────────────────────────────────────────┐    │
│  │  ~/wiki/              個人筆記 (Karpathy Wiki)           │    │
│  │  graphify-out/        專案知識圖譜 (Graphify)            │    │
│  │  Jira comments        跨 session 持久化記憶              │    │
│  └──────────────────────────────────────────────────────────┘    │
│         │                                                        │
│         ▼                                                        │
│  ┌─ 外部系統 ──────────────────────────────────────────────┐    │
│  │  Jira (Atlassian MCP)    ticket + 計畫 + summary         │    │
│  │  GitHub (gh CLI)         PR + commit                      │    │
│  └──────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 知識系統

系統使用兩套知識庫,分別處理「你學到的東西」和「專案程式碼的結構」:

```
┌─────────────────────────────────────────────────────────┐
│                    知識系統                               │
├─────────────────────┬───────────────────────────────────┤
│  個人筆記 (Wiki)     │  專案程式碼 (Graphify)             │
│  ~/wiki/             │  專案/graphify-out/                │
│                      │                                   │
│  手動整理 + AI 維護   │  AI 自動掃描產生                   │
│  Karpathy Wiki 模式  │  知識圖譜模式                      │
├─────────────────────┼───────────────────────────────────┤
│  用 /wiki 操作       │  用 /graphify 操作                 │
│  (在 Claude Code)    │  (在 Claude Code)                 │
└─────────────────────┴───────────────────────────────────┘
         │                         │
         └────────┬────────────────┘
                  ▼
         /plan-to-jira Layer 1.5
         /ship Phase 2.1
         兩個都會同時讀兩邊
```

### 兩套系統的差異

| | 個人筆記 Wiki | 專案 Graphify |
|---|---|---|
| **位置** | `~/wiki/` | `專案目錄/graphify-out/` |
| **來源** | 你的學習紀錄、文章、白板 | 專案的原始碼(.go 檔案) |
| **產生方式** | `/wiki`(你丟資料進去,AI 整理) | `/graphify .`(AI 自動掃描整個 codebase) |
| **內容** | AI Agent 概念、AWS、Backend 知識等 | 程式碼架構、模組關聯、函式依賴 |
| **格式** | markdown wiki + `index.md` | 知識圖譜 `graph.json` + markdown wiki |
| **更新** | 有新資料就 `/wiki` | 程式碼改了就 `/graphify . --update` |
| **查詢** | `/wiki query "問題"` | Claude Code 自動讀(有 PreToolUse hook) |
| **Token 效率** | 直接讀原檔 | **71x 更省 token**(知識圖譜壓縮) |

### 個人筆記 Wiki 目錄結構(Karpathy Wiki 模式)

```
~/wiki/
├── CLAUDE.md          # Schema — 定義 AI 的行為規則
├── index.md           # 內容目錄:每頁一行摘要
├── log.md             # 操作日誌:append-only
├── raw/               # 原始資料(唯讀,AI 只讀不改)
│   ├── whiteboard/    # 白板照片 / 心智圖
│   ├── cards/         # Heptabase 卡片匯入
│   └── journal/       # 日記 / 工作日誌
└── wiki/              # AI 撰寫與維護的知識頁面
    ├── AI/            # Agent、MCP、RAG、Prompt 等
    ├── AWS/           # ECS、Lambda、SQS 等
    ├── Backend/       # Google Identity Services、NoSQL 等
    └── {{PROJECT}}/   # 專案相關知識
```

**三層原則**:
- `raw/`:你的原始素材,**永不修改**
- `wiki/`:AI 撰寫、維護、互相 `[[連結]]` 的知識頁面
- `CLAUDE.md`:規範 AI 如何操作這一切

### Graphify 知識圖譜(專案程式碼)

**安裝**:
```bash
uv tool install graphifyy              # 安裝 Graphify CLI
graphify install --platform claude      # 安裝 Claude Code skill
```

**使用**(全部在 Claude Code 裡面執行,不是 terminal):
```bash
/graphify .                  # 首次掃描 — 建立知識圖譜
/graphify . --wiki           # 首次掃描 + 產出 markdown wiki
/graphify . --update         # 程式碼改了之後增量更新
/graphify query "問題"       # 基於圖譜查詢
```

**掃描完後產出**:
```
專案目錄/graphify-out/
├── graph.json              # 知識圖譜(結構化,NetworkX 格式)
├── GRAPH_REPORT.md         # 架構摘要(god nodes、community 分群)
└── wiki/
    ├── index.md            # wiki 索引(agent 導航入口)
    └── community_*.md      # 各模組/社群的 wiki 頁面
```

**運作原理**:
1. **AST 解析**(不耗 token):用 Tree-sitter 解析所有 `.go` 檔的函式、結構體、依賴關係
2. **語義提取**(耗 token):用 LLM subagent 並行處理文檔、註釋、README
3. **圖譜合成**:合併為 NetworkX 圖,用 Leiden 演算法做社群偵測
4. **輸出**:互動式 HTML + 可查詢的 JSON + markdown wiki

**與 `/plan-to-jira` 和 `/ship` 的整合**:

Graphify 安裝後會在 `.claude/settings.json` 加一個 PreToolUse hook — 每次 Claude Code 要用 Glob/Grep 搜尋檔案前,會先提醒「知識圖譜存在,先讀 GRAPH_REPORT.md」。這讓 agent 不需要盲目搜尋,而是從圖譜導航。

---

## Commands 完整清單

所有指令都在 **Claude Code 裡面**執行(不是 terminal)。

### 開發工作流

| 指令 | 做什麼 | 說明 |
|------|--------|------|
| `/plan-to-jira {{JIRA_PROJECT_KEY}}-XXX 描述` | 三層確認 → 規劃 → 寫 Jira | 規劃階段,產出計畫 |
| `/ship {{JIRA_PROJECT_KEY}}-XXX` | 端到端:規劃→實作→審查→測試→PR | 執行階段,照計畫做到底 |

### 知識管理

| 指令 | 做什麼 | 說明 |
|------|--------|------|
| `/wiki` | 將當前研究結果寫入個人 wiki | ingest 模式 |
| `/wiki <主題>` | 同上,指定主題 | |
| `/wiki query "問題"` | 從個人 wiki 查詢並回答 | 有價值的答案會回寫 wiki |
| `/wiki lint` | 健診 wiki 完整性 | 找矛盾、孤頁、缺失連結 |
| `/graphify .` | 首次掃描專案,建立知識圖譜 | 會耗 token(語義提取) |
| `/graphify . --wiki` | 掃描 + 產出 markdown wiki | 讓 /plan-to-jira 和 /ship 能讀 |
| `/graphify . --update` | 增量更新圖譜(只處理改過的檔案) | 比全量掃描快 |

### 背後的 Agents(自動派出,你不需要手動叫)

| Agent | subagent_type | Model | 做什麼 | 誰派的 |
|-------|--------------|-------|--------|--------|
| Planner | `planner` | **opus** | 讀 codebase 產計畫 | `/ship` `/plan-to-jira` |
| Code Reviewer | `go-reviewer` | **sonnet** | 程式碼品質審查 | `/ship` Phase 3 |
| Security Reviewer | `go-security-reviewer` | **sonnet** | Go 安全漏洞檢查 | `/ship` Phase 3 |
| Build Resolver | `go-build-resolver` | **sonnet** | 修 build 錯誤 | build 失敗時 |
| Test Writer | `go-tdd-guide` | **sonnet** | 寫 unit test | `/ship` Phase 3 |
| Fix Applier | `go-reviewer` | **sonnet** | 套用 CRITICAL/HIGH 修正 | review 後 |
| Critique Agent | `general-purpose` | **sonnet** | 唱反調質疑計畫 | `/plan-to-jira` |
| 4x Doc Writers | `general-purpose` | **haiku** | 並行產 Jira 文件 | `/ship` Phase 4 |

**圖例**:∥ = 並行(read-only),→ = 依序(write,必須等前一個完成)

### 背後的 Skills(被 commands 呼叫,你不直接用)

| Skill | 被誰用 | 用途 |
|-------|--------|------|
| `agent-io` | `/ship` | 6 種標準化 agent IO 合約 |
| `jira-read` | `/ship` Phase 1 | 解析 Jira ticket |
| `jira-write-bug` | `/ship` Phase 4 | Bug 報告 Jira 格式 |
| `jira-write-task` | `/ship` Phase 4 | Task/Story Jira 格式 |
| `api-change-doc` | `/ship` Phase 4 | API 變更文件格式 |
| `storage-change-doc` | `/ship` Phase 4 | DB/儲存層變更文件 |
| `integration-test` | `/ship` Phase 3.5 | curl-based API 整合測試 |
| `git-cz-commit` | `/ship` Phase 6 | Conventional commit |
| `pr-create` | `/ship` Phase 7 | PR 建立(含 template) |
| `swagger-conventions` | `/ship` Phase 3 | Swagger 註解規範 |

---

## 兩個核心命令

### `/plan-to-jira` — 規劃者(Planner)

把需求轉成結構化實作計畫,寫入 Jira。

**流程:**
1. 解析 Jira ticket key + 描述
2. **三層確認**(商業邏輯 → 技術思路 → code 細節)
3. **Wiki + Graphify 自動載入**:Layer 1 確認後自動搜尋個人筆記 + 專案知識圖譜
4. 派出 **planner agent** 讀 codebase 產出計畫
5. 派出**批判者 agent**(唱反調)質疑計畫
6. 格式化為雙受眾格式(人看的 + agent 用的)
7. 寫入 Jira comment

**設計亮點:**
- **三層確認**防止直接跳到 code 而不理解商業問題
- **字數上限**(300/500字)是自檢機制:說不清楚代表你還不懂
- **批判者 agent** 有反偷懶規則:「看起來沒問題」不是有效結論
- **雙受眾格式**:上半部給 PM/工程師看,下半部給實作 agent 照做

```bash
/plan-to-jira {{JIRA_PROJECT_KEY}}-18000 遷移 Google 登入到 GIS
```

### `/ship` — 執行者 + 驗證者(Generator + Evaluator)

端到端開發 orchestrator:規劃 → 實作 → 審查 → 測試 → 文件 → 提交 → PR。

**流程:**
1. 讀 Jira ticket
2. 檢查 Jira 上是否已有 `/plan-to-jira` 寫的計畫(跨 session 狀態)
3. 收集 Wiki + Graphify context
4. 沒有計畫就從頭規劃(帶入知識背景)
5. Sanity check 驗證計畫中的檔案路徑都存在
6. **建立 git worktree**(隔離工作目錄,不影響主 repo)
7. 派出 implementation agent 實作(在 worktree 裡)
8. Build 驗證
9. **並行**派出 code review + security review agent
10. **依序**套用修正
11. 寫 unit tests
12. 跑 integration tests
13. **並行**派出 4 個 haiku agent 產文件
14. 寫 summary 到 Jira
15. Conventional commit + Push(在 worktree 裡)
16. 建 PR
17. 清理 worktree

**設計亮點:**
- **Orchestrator 不寫 code** — 只 dispatch agent 和把關確認
- **Read agent 並行跑,write agent 依序跑**(避免 merge 衝突)
- **Git worktree 隔離** — 實作在獨立目錄進行,你的主 repo 不受影響
- **Rollback trigger**:CRITICAL issues > 3 時建議回到規劃階段,而非硬修
- **5 個確認 gate**:每個重要環節人工把關

```bash
/ship {{JIRA_PROJECT_KEY}}-18000
```

---

## 兩個命令怎麼串接

```
/plan-to-jira                            /ship
┌──────────────┐    Jira comment      ┌──────────────┐
│ 三層確認      │   "## Impl Plan"    │ Phase 2.0    │
│ → Wiki 載入  │ ───────────────────▶ │ 讀 Jira 計畫 │
│ → 計畫       │   跨 session        │ → 直接用     │
│ → 批判者審查  │   持久化記憶         │ → 實作       │
│ → 寫 Jira    │                      │ → 審查+測試  │
└──────────────┘                      └──────────────┘
```

Jira 作為**跨 session 的持久化記憶**。`/plan-to-jira` 寫的計畫,在完全不同的 session 被 `/ship` 撿起來直接用。

---

## 完整流程圖

### `/plan-to-jira`

```
用戶輸入: /plan-to-jira {{JIRA_PROJECT_KEY}}-XXXXX 描述
│
▼
┌─ Step 1: 解析輸入 ────────────────────────────────────────┐
│  提取 ticket key ({{JIRA_PROJECT_KEY}}-\d+) + 計畫描述    │
│  缺一不可,缺了就問用戶                                     │
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Step 2: 三層確認(強制,不可跳過)─────────────────────────┐
│                                                            │
│  Layer 1: 商業邏輯確認(≤300 字)                           │
│  ├─ 問題 / 影響對象 / 成功標準 / Scope 邊界                │
│  ├─ 附好壞範例(Few-shot)                                  │
│  ├─ 超過 300 字 = 你不清楚 → 必須問用戶                    │
│  └─ ★ STOP 等 ok                                    Gate 1│
│                                                            │
│  Layer 1.5: Wiki + Graphify 自動載入(靜默執行)            │
│  ├─ 搜尋 ~/wiki/ 是否有相關頁面                             │
│  ├─ 搜尋 graphify-out/ 是否有相關架構資訊                   │
│  └─ 合併作為 Layer 2 背景知識                               │
│                                                            │
│  Layer 2: 技術思路確認(≤500 字)                           │
│  ├─ Before / After / 做法 / 不確定的部分                    │
│  ├─ 附好壞範例(Few-shot)                                  │
│  ├─ 超過 500 字 = 思路不清晰 → 必須問用戶                  │
│  └─ ★ STOP 等 ok                                    Gate 2│
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Step 3: Planner Agent (opus) ────────────────────────────┐
│  讀 CLAUDE.md → 探索 codebase → 產出結構化計畫              │
│  用戶可反覆修改直到滿意                                     │
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Step 3.5: 計畫批判者(唱反調)── sonnet agent ───────────┐
│  角色:「你的任務不是確認計畫能不能用,而是找出問題」       │
│  ├─ 驗證所有路徑存在(Glob/Grep)                          │
│  ├─ 檢查:漏改檔案、依賴順序、過度設計、edge case          │
│  ├─ 反偷懶規則:「看起來沒問題」不是有效結論                │
│  ├─ 輸出:Issue + Severity + Evidence + Suggestion          │
│  ├─ CRITICAL/HIGH → 回 Step 3 修改(迴圈)                 │
│  └─ MEDIUM/LOW → FYI 呈現,繼續                            │
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Step 4-7: 確認 → 格式化 → 寫 Jira ─────────────────────┐
│  ★ STOP 等 yes/no                                  Gate 3│
│  雙受眾格式(人看的 + agent 用的)                          │
│  產出:Jira comment "## Implementation Plan"               │
└───────────────────────────────────────────────────────────┘
```

### `/ship`

```
用戶輸入: /ship {{JIRA_PROJECT_KEY}}-XXXXX
│
▼
┌─ Phase 1: 讀 Jira ── jira-read skill ────────────────────┐
│  ★ STOP 等確認                                     Gate 1│
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Phase 2: 規劃 ──────────────────────────────────────────┐
│  2.0 檢查 Jira 是否已有計畫(跨 session 狀態)            │
│  ├─ 找到 → "要用這個計畫嗎?"                              │
│  └─ 沒找到 → 從頭規劃                                     │
│                                                            │
│  2.1 收集 Wiki + Graphify context                          │
│  2.2 Planner agent (opus) — 帶入知識背景                   │
│  2.3 Sanity check:驗證所有檔案路徑存在                    │
│  2.4 呈現計畫(含警告)                                    │
│  ★ STOP 等確認                                     Gate 2│
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Phase 2.5: 建立 Git Worktree ───────────────────────────┐
│  git worktree add → 隔離工作目錄 + feature branch          │
│  從這裡開始所有 agent 都在 worktree 裡工作                  │
│  你的主 repo 完全不受影響                                   │
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Phase 3: 實作 + 驗證(在 worktree 裡)─────────────────┐
│  3.1 Implementation (sonnet, write) →                      │
│  3.2 Build verification →                                  │
│  3.3 Code review (sonnet, read) ─┐ ★ 並行                 │
│  3.4 Security review (sonnet) ───┘                         │
│  3.3→fix Apply fixes (sonnet, write) →                     │
│  3.4 Write tests (sonnet, write) →                         │
│  3.5 Integration tests →                                   │
│  3.6 Quality gates + rollback trigger                      │
│      CRITICAL > 3 → Re-plan / Fix forward / Abort         │
│  ★ STOP 等確認                                     Gate 3│
└───────────────────────────────────────────────────────────┘
│
▼
┌─ Phase 4: 文件生成 ── 4 個 haiku agent 並行 ─────────────┐
│  4.2 Summary ────────┐                                     │
│  4.3 API 變更 ───────├─ ★ 並行                             │
│  4.4 儲存層變更 ─────┤                                     │
│  4.5 流程圖 ─────────┘                                     │
│  4.6 組裝(不用 AI,直接拼接)                              │
│  ★ STOP 等確認                                     Gate 4│
└───────────────────────────────────────────────────────────┘
│
▼
Phase 5: 寫 Jira
Phase 6: Commit + Push(在 worktree 裡)
Phase 7: PR                                       ★ Gate 5│
Phase 8: 清理 worktree
```

---

## 目錄結構

```
claude-code-harness/
├── agents/                  # Agent 定義檔(subagent_type 對應的角色設定)
│   ├── planner.md           # 規劃 agent — 讀 codebase 產出實作計畫
│   ├── go-reviewer.md       # Go 程式碼審查 agent
│   ├── go-build-resolver.md # Go build 錯誤修復 agent
│   ├── go-security-reviewer.md # Go 安全漏洞檢查 agent
│   └── go-tdd-guide.md      # Go 單元測試撰寫 agent
├── commands/                # Slash commands(用戶直接呼叫的工作流)
│   ├── plan-to-jira.md      # /plan-to-jira — 規劃工作流
│   └── ship.md              # /ship — 端到端開發工作流
├── skills/                  # 子 Skills(被 commands 呼叫,用戶不直接用)
│   ├── agent-io.md          # Agent 輸入/輸出合約(6 種標準格式)
│   ├── jira-read.md         # 讀取並解析 Jira ticket
│   ├── jira-write-bug.md    # Jira Bug 報告格式
│   ├── jira-write-task.md   # Jira Task/Story 格式
│   ├── api-change-doc.md    # API 變更文件格式
│   ├── storage-change-doc.md # DB/儲存層變更文件格式
│   ├── integration-test.md  # curl-based API 整合測試
│   ├── git-cz-commit.md     # Conventional commit 工作流
│   ├── pr-create.md         # PR 建立(含 template)
│   └── swagger-conventions.md # Swagger 註解規範
├── rules/                   # 全域規則(每個 session 自動載入)
│   ├── common/              # 語言無關的通用規則
│   └── golang/              # Go 專屬規則
└── README.md                # 本文件
```

---

## Skills 參考

### Rules(全域規則,每個 session 自動載入)

| 規則 | 檔案 | 用途 |
|------|------|------|
| agents | `rules/common/agents.md` | Agent 編排:什麼時候用哪個 agent |
| coding-style | `rules/common/coding-style.md` | 不可變性、檔案組織、錯誤處理 |
| git-workflow | `rules/common/git-workflow.md` | Commit 格式、PR 流程 |
| hooks | `rules/common/hooks.md` | Pre/PostToolUse hooks、TodoWrite |
| patterns | `rules/common/patterns.md` | Repository pattern、API response 格式 |
| performance | `rules/common/performance.md` | Model 選擇、context window 管理 |
| security | `rules/common/security.md` | 安全檢查清單、secret 管理 |
| testing | `rules/common/testing.md` | TDD 流程、80% 覆蓋率 |

---

## Agent 定義檔設計說明

Agent 定義檔放在 `agents/` 目錄,每個檔案定義一個 `subagent_type` 的角色、工具、行為規則。當 command 或 orchestrator 用 `Agent` tool dispatch 時,Claude Code 會載入對應的 agent 定義作為 system prompt。

### 每個 Agent 的共通結構

```
┌─ Frontmatter ─────────────────────────────────┐
│  name, description, tools, model               │  ← Tool Loadout + Model Tiering
└────────────────────────────────────────────────┘
┌─ MUST DO FIRST ───────────────────────────────┐
│  1. 讀 CLAUDE.md                               │  ← 先看再改
│  2. 任務特定的初始動作                          │
└────────────────────────────────────────────────┘
┌─ 主體內容 ────────────────────────────────────┐
│  角色定義、流程、檢查清單、輸出格式             │  ← 嚴格定義輸出格式
└────────────────────────────────────────────────┘
┌─ Critical Rules(NEVER + Why)────────────────┐
│  禁令 + 原因,設定行為底線                      │  ← 禁令代替指令 + 禁令附原因
└────────────────────────────────────────────────┘
```

### 各 Agent 使用的設計技巧

#### `planner.md` — 規劃 Agent

| 區塊 | 設計技巧 | 說明 |
|------|---------|------|
| `tools: ["Read", "Grep", "Glob"]` | **Tool Loadout** | 只給讀取工具,物理上防止 planner 改 code |
| `model: opus` | **Model Tiering** | 規劃需要深度推理,用最強模型 |
| MUST DO FIRST: 讀 CLAUDE.md | **先看再改** | 確保專案慣例在 agent 開始前就載入 |
| "If unclear, **STOP and ask**" | **要求 AI 先反問** | 強制規則,不是可選的 "ask if needed" |
| "Define scope boundary: IN and OUT" | **不要畫蛇添足** | 防止 planner 自己擴大需求範圍 |
| 架構層級(api/module/lib) | **專案專屬知識** | agent 知道 package prefix 和 clean architecture 結構 |
| "Exact file paths — verified by Glob" | **語境中毒防護 (CoVe)** | 防止 planner 幻覺出不存在的路徑 |
| "schema → domain → repository → usecase → handler" | **Chain of Thought** | 強制按 clean architecture 依賴方向排序 |
| Plan Format + Scope Boundary | **嚴格定義輸出格式** | 統一格式讓下游 agent 能直接解析 |
| Worked Example | **Few-shot** | 用具體 Go 範例,不是通用 TypeScript 範例 |
| Sizing and Phasing(4 phase) | **Least-to-Most** | 大問題拆成獨立可交付的小階段 |
| Red Flags to Check | **Reflection** | planner 產出後自我審查 |
| 6 條 NEVER + Why | **禁令 + 原因** | 設定底線並解釋原因,讓 AI 理解而非盲從 |
| When You Don't Know(4 種情境) | **不知道就說不知道** | 明確列出必須停下來問的情境 |

#### `go-reviewer.md` — 程式碼審查 Agent

| 區塊 | 設計技巧 | 說明 |
|------|---------|------|
| CRITICAL/HIGH/MEDIUM 分級 | **嚴格定義輸出格式** | 用嚴重度分級,不是含糊的「建議」 |
| HIGH — 專案 Conventions 區塊 | **專案專屬知識** | import grouping、命名、函式長度限制 |
| `cCopy` in goroutine check | **專案慣例** | gin.Context in goroutine 規則 |
| Approval Criteria(Approve/Warn/Block) | **Agent IO Contracts** | 標準化輸出讓 orchestrator 能判斷是否需要 fix agent |
| 4 條 NEVER(含:不 flag CLAUDE.md 允許的寫法) | **禁令 + 原因** | 防止 reviewer 跟專案慣例打架 |

#### `go-build-resolver.md` — Build 修復 Agent

| 區塊 | 設計技巧 | 說明 |
|------|---------|------|
| "Surgical fixes only" | **不要畫蛇添足** | 只修 build error,不改 style,不重構 |
| "NEVER read files beyond what's needed" | **少讀檔案** | 用 Grep 定位確切行數,不讀整個檔案 |
| Stop Conditions(3 次嘗試後停) | **失敗模式防護** | 防止無限迴圈修不好的 error |
| Common Fix Patterns 表格 | **Few-shot** | 常見錯誤 → 原因 → 修法,加速診斷 |

#### `go-security-reviewer.md` — 安全審查 Agent

| 區塊 | 設計技巧 | 說明 |
|------|---------|------|
| OWASP Top 10 Go 版 | **語言專屬知識** | GORM injection、cCopy、SELECT FOR UPDATE |
| Code Pattern Checks 表格 | **嚴格定義輸出格式** | 具體 grep pattern,不是模糊描述 |
| Diagnostic Commands | **可執行的檢查** | 給 agent 實際能跑的指令 |
| "NEVER dismiss without evidence" | **反偷懶** | 跟 critique agent 同理 |
| "ALWAYS check for IDOR" | **失敗模式防護** | IDOR 是最常被漏掉的安全問題 |
| "Do NOT review general web security that doesn't apply to Go" | **不要畫蛇添足** | 防止報告 JS/DOM 相關的無關問題 |

#### `go-tdd-guide.md` — 測試撰寫 Agent

| 區塊 | 設計技巧 | 說明 |
|------|---------|------|
| "Do NOT rely on implementation agent's summary" | **Context Isolation** | 獨立判斷,不被實作 agent 的摘要偏誤影響 |
| `//go:build unit` + testify + table-driven | **專案專屬知識** | build tag 是必須的,testify 是團隊標準 |
| NO AAA comments、`t.Parallel()`、`wantErr error` | **團隊慣例** | 來自用戶明確的 feedback |
| 完整 table-driven test 模板 | **Few-shot** | 可直接複製的模板 |
| Must NOT Test 清單 | **不要畫蛇添足** | 不測 private function、不測 GORM、不測 generated code |
| "NEVER read mock files" + `go generate` | **團隊慣例** | mock 是自動產生的,讀了浪費 context |

### 技巧覆蓋矩陣

| 技巧 | planner | go-reviewer | go-build-resolver | go-security-reviewer | go-tdd-guide |
|------|:-------:|:-----------:|:-----------------:|:--------------------:|:------------:|
| 先看再改(先讀 CLAUDE.md) | ✓ | ✓ | ✓ | ✓ | ✓ |
| 禁令 + 原因(NEVER + Why) | 6 條 | 4 條 | 5 條 | 5 條 | 5 條 |
| 不知道就問 | ✓ | — | — | — | — |
| 不要畫蛇添足(scope) | ✓ | — | ✓ | ✓ | ✓ |
| 專案專屬知識 | 架構 | 命名 | — | GORM/cCopy | build tag/mockery |
| Few-shot 範例 | Go 範例 | — | 表格 | — | table-driven |
| 嚴格輸出格式 | Plan Format | severity | FIXED format | severity | summary |
| Tool Loadout | 只讀 | +Bash | +Write/Edit | 只讀 | +Write/Edit |
| Model Tiering | opus | sonnet | sonnet | sonnet | sonnet |
| 反偷懶 | — | — | — | ✓ | — |
| Reflection 自檢 | Red Flags | — | Stop Conditions | — | — |
| Context Isolation | — | — | — | — | ✓ |
| 語境中毒防護 | Glob 驗證 | — | — | — | — |
| 失敗模式防護 | — | — | 3 次停止 | IDOR check | — |

---

## 使用的技術

### Agent 架構(21 項)

| 技術 | 怎麼用的 | 在哪裡 | 對應理論 |
|------|---------|--------|---------|
| Orchestrator-Worker | main thread 只 dispatch,不自己做事 | `/ship` | 監督者模式 |
| Context Isolation | 傳 pointer 不傳 content,每個 agent 只看自己需要的 | `/ship` | 上下文隔離 |
| Concurrency Control | read ∥ 並行,write → 依序 | `/ship` Phase 3 | 並行監督者 + 順序流 |
| KV Cache Optimization | prompt 靜態部分放前面,動態放後面 | `/ship` 所有 agent | KV Caching 工程技巧 |
| Model Tiering | opus→編排, sonnet→寫 code, haiku→文件 | `/ship` | 成本分級(師生式協同) |
| Agent IO Contracts | 6 種標準化輸入/輸出格式 | `/ship` | 結構化通訊協議 |
| 3A 架構 | Planner → Generator → Evaluator | 兩者皆有 | Planner-Generator-Evaluator |
| Exec-Review-Adjust | sanity check 驗證 planner 結果 | `/ship` Phase 2.2 | 審查機制 |
| Rollback Trigger | CRITICAL > 3 → 建議回到 Phase 2 | `/ship` Phase 3.6 | 回滾機制 |
| Human-in-the-Loop | 5 gates (ship) + 3 gates (plan-to-jira) | 兩者皆有 | 關鍵決策點人工審核 |
| 禁令 + 原因 | 每條 NEVER 都附上 why | `/ship` | 禁令要寫清楚為什麼 |
| Skill 封裝 | jira-read、integration-test 等都是黑盒子 | `/ship` | 把 SOP 封裝為 Skill |
| 動態工具曝露 | Phase 4.1 先讀 skill 內容,再 inline 給 agent | `/ship` Phase 4 | 信息按需供給 |
| 先看再改 | 所有 agent 都要先讀 CLAUDE.md | 兩者皆有 | 先讀後改原則 |
| 一次授權 ≠ 永久授權 | 每個 phase 都重新等 gate | 兩者皆有 | 授權範圍管控 |
| 不要畫蛇添足 | scope constraints 限制 agent 只做自己的事 | `/ship` | 不加沒被要求的功能 |
| 工作分出去,思考不行 | orchestrator 自己消化結果再 dispatch | `/ship` | 不外包理解力 |
| 唱反調角色 | critique agent 專門找問題,不准確認 | `/plan-to-jira` | 生成與評估分離 |
| 反偷懶話術 | 「看起來沒問題」不是有效結論 | `/plan-to-jira` | Anthropic 源碼反駁模式 |
| 跨 Session 狀態 | Jira comment 作為持久化記憶 | `/ship` Phase 2.0 | 記憶持久化 |
| 外部系統整合 | Atlassian MCP, GitHub CLI | 兩者皆有 | 工具編排 |

### Prompt 工程(10 項)

| 技術 | 怎麼用的 | 在哪裡 | 對應理論 |
|------|---------|--------|---------|
| 嚴格定義輸出格式 | 6 種 Agent IO Contracts + Jira 8 段結構 | 兩者皆有 | 手術級精準修改 |
| 要求 AI 先反問 | 資訊不足就停,用 AskUserQuestion 問 | `/plan-to-jira` | 防堵錯誤假設 |
| Chain of Thought | 強制思考順序:商業→技術→code | `/plan-to-jira` | 展開推理鏈 |
| Few-shot(好壞範例) | Layer 1/2 都有好範例和壞範例 | `/plan-to-jira` | 上下文學習 |
| 字數上限 = 自檢 | 超字數 = 你不清楚,必須先問 | `/plan-to-jira` | 約束邊界防模糊 |
| 禁令代替指令 | NEVER/DO NOT 設底線 | 兩者皆有 | 紅線思維 |
| 禁令附原因 | 每條 NEVER 附 why | `/ship` | 讓 AI 理解而非盲從 |
| 雙受眾格式 | 上半部人看,下半部 agent 用 | `/plan-to-jira` | 信息分層 |
| 不知道就說不知道 | 資訊不足就停,絕不假設 | `/plan-to-jira` | 承認無知 |
| Step-back Prompting | 先抽象(商業)→ 再具體(技術) | `/plan-to-jira` | 先抽象後具體 |

### Context 管理(8 項 + 4 防護)

| 技術 | 怎麼用的 | 在哪裡 | 對應理論 |
|------|---------|--------|---------|
| RAG | 傳 pointer,agent 自己去讀 | `/ship` | 精挑細選 > 海量傾倒 |
| Tool Loadout | deferred tools,用到才載入 schema | `/ship` | 工具 < 30 保持準確 |
| Context Quarantine | 每個 agent 獨立 context window | `/ship` | 無菌室工作 |
| Context Pruning | "Receive summaries, not dumps" | `/ship` | 剪掉廢話 |
| Context Summarization | Phase 3.6 摘要;Jira 跨 session | 兩者皆有 | 精華 > 舊帳 |
| 隔離 | 只傳相關數據給子 agent | `/ship` | LangChain 策略 1 |
| 持久化 | Jira comment 跨 session 記憶 | 兩者皆有 | LangChain 策略 2 |
| 壓縮 | Phase 3.6 只傳 summary 不傳原始 review | `/ship` | LangChain 策略 3 |

**Context 問題防護:**

| 問題 | 防護 | 在哪裡 |
|------|------|--------|
| 語境中毒(幻覺進入上下文) | sanity check + 要求附 evidence | `/ship` 2.2 + `/plan-to-jira` 3.5 |
| 語境干擾(上下文壓倒訓練) | 每個 agent 乾淨 context window | `/ship` |
| 語境混淆(多餘上下文) | 傳 pointer 不傳 content | `/ship` |
| 語境衝突(上下文內部矛盾) | write agent 依序跑,同一時間只有一個改 code | `/ship` Phase 3 |

### Harness Engineering 六層架構(全部覆蓋)

| 層 | 內容 | 實作 | 狀態 |
|----|------|------|------|
| 1. 信息邊界 | 角色隔離 + 信息裁切 | CLAUDE.md + scope constraints + context isolation | ✅ |
| 2. 工具系統 | 動態曝露 + 結果提煉 | deferred tools + agent IO contracts + model tiering | ✅ |
| 3. 執行編排 | 鏈路設計 + Checklist | 8 phase pipeline + concurrency rules + KV cache + worktree | ✅ |
| 4. 記憶與狀態 | 分類管理 + 狀態快照 | Jira 跨 session + Wiki + Graphify + Phase 間 summary | ✅ |
| 5. 評估與觀測 | 生產驗收分離 | code review + security review + critique agent + go build/vet | ✅ |
| 6. 約束與恢復 | 重試 + 糾偏 + 回滾 | build resolver + rollback trigger + 5 gate system | ✅ |

### 17 種 Agent 架構(覆蓋 12/17)

| 階梯 | 技術 | 用了嗎 | 在哪裡 | 備註 |
|------|------|--------|--------|------|
| 第一階 | Zero-shot | ❌ | — | 每個 prompt 都有 instructions |
| | Few-shot | ✅ | `/plan-to-jira` 好壞範例 | |
| | CoT | ✅ | `/plan-to-jira` Layer 1→2 | |
| 第二階 | Self-consistency | ❌ | — | 非數學推理,不需要 |
| | ToT | ❌ | — | 同上 |
| | GoT | ❌ | — | 同上 |
| 第三階 | Least-to-Most | ✅ | `/plan-to-jira` 拆解需求 | |
| | Step-back | ✅ | Layer 1 抽象 → Layer 2 具體 | |
| | Plan-and-Solve | ✅ | 整個 `/plan-to-jira` | |
| 第四階 | ReAct | ✅ | 每個 agent 底層 Think→Act→Observe | |
| | Reflection | ✅ | critique agent + rollback trigger | |
| | CoVe | ✅ | sanity check 驗證 planner 結果 | |
| 第五階 | Generated Knowledge | ✅ | planner 先讀 codebase + Graphify 再產計畫 | |
| | Self-RAG | ✅ | agent 自己決定讀哪些檔案 | |
| | CRAG | ❌ | — | 不需要網頁搜尋 |
| 第六階 | MemGPT | ⚠️ | Jira + Wiki + Graphify 持久化 | 非 agent 自主管理 |
| | Multi-Agent Manager-Worker | ✅ | `/ship` orchestrator-worker | |

### 多智能體協作模式

| 模式 | 用了嗎 | 在哪裡 | 備註 |
|------|--------|--------|------|
| 上下級協同(Hierarchical) | ✅ | `/ship` orchestrator → agents | 核心架構 |
| 師生式(Master-Disciple) | ✅ | `/ship` Phase 4:opus 給標準,haiku 產草稿 | 成本優化 |
| 競爭式(Competitive) | ❌ | — | 工程有唯一正確答案 |

### 失敗模式防護

| 問題 | 防護 | 在哪裡 |
|------|------|--------|
| 無聲失敗(數據看似正確) | sanity check + critique 要求 evidence | `/ship` 2.2 + `/plan-to-jira` 3.5 |
| 規格偏移(Spec Drift) | 8 個 confirmation gates | 兩者皆有 |
| 順從性偏誤(Sycophancy) | 反偷懶規則 | `/plan-to-jira` 3.5 |
| 級聯失敗 | Human-in-the-Loop + rollback trigger | `/ship` |

---

## Agent 成長路徑定位

```
Level 1: API Call              ██████████ 已超越
Level 2: Workflow              ██████████ 已超越
Level 3: 對話式 Agent          ██████████ 已超越
Level 4: 上下文隔離 + 記憶體   ██████████ ← 目前位置(最高階)
```

---

## 如何根據自己的專案調整

1. **替換占位符**:用 [快速開始](#快速開始專案占位符) 表格中的值全檔搜尋取代
2. **替換 `CLAUDE.md`**:用你自己的專案規範(放在 `$REPO_ROOT/CLAUDE.md`)
3. **替換 Jira 設定**:換成你的 issue tracker(或拿掉)
4. **替換 Go 專屬 agent**:`go-reviewer`、`go-build-resolver` 換成你的語言對應版本
5. **調整 model tiering**:根據預算和任務複雜度
6. **設定知識系統**:
   - 個人筆記:建立 `~/wiki/` + `CLAUDE.md` schema + `/wiki` command
   - 專案圖譜:`uv tool install graphifyy` + `graphify install --platform claude` + `/graphify .`
7. **保留架構 pattern**:以下跟語言無關,可以直接用:
   - Orchestrator-Worker(read/write 分離)
   - 3A(Planner → Generator → Evaluator)
   - Human-in-the-Loop gates
   - 跨 session 狀態用外部系統
   - Git worktree 隔離實作

---

## 參考資料

- [Anthropic Claude Code 文件](https://docs.anthropic.com/en/docs/claude-code)
- [LangChain: Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/)
- [Harness Engineering 概念](https://www.youtube.com/watch?v=GDm_uH6VxPY)
- [Karpathy LLM Wiki 模式](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) — 個人筆記 Wiki 的核心概念
- [Graphify — 知識圖譜工具](https://github.com/safishamsi/graphify) — 專案程式碼 → 知識圖譜

---

## License

MIT
