# Ship: Jira-Driven Development Workflow

End-to-end workflow orchestrator. The main thread acts as **orchestrator only** — it sequences phases, prepares agent prompts, and gates user confirmation. It does NOT read large codebases or write implementation code directly.

## Orchestrator Principles

### Role
The main thread is a **dispatcher**. It:
- Reads Jira cards and user input (lightweight)
- Prepares structured prompts for agents (with file paths, not file contents)
- Dispatches agents with explicit instructions
- Collects results and presents summaries
- Gates user confirmation between phases

### Agent Prompt Rules
Every agent dispatch MUST include a structured prompt with:
1. **Task objective**: One-line summary of what to do
2. **File paths**: `git diff --name-only` output — NEVER paste file contents (file contents bloat the orchestrator's context window and break context isolation between agents)
3. **Conventions reference**: "Read `$REPO_ROOT/CLAUDE.md` for project conventions"
4. **Scope constraints**: What to touch, what NOT to touch
5. **Output format**: What the orchestrator expects back (e.g., "Return a summary of issues found with severity levels")

### Concurrency Rules
- **Read-only agents CAN run in parallel** (code review, security review, analysis)
- **Write agents MUST run sequentially** (only ONE agent writes code at a time)
- If an agent modifies code, subsequent agents must run AFTER it completes to see the updated state

### Data Passing
- **Pass pointers, not content**: file paths, line ranges, function names
- **Pass skills, not ad-hoc instructions**: reference specific skills (e.g., "Follow Go testing conventions from CLAUDE.md (testify, table-driven, `//go:build unit`) patterns") so agents follow consistent standards
- **Receive summaries, not dumps**: instruct agents to return concise results (pass/fail, issue count, coverage %)

### Prompt Ordering (KV Cache Optimization)
All agent prompt templates MUST follow this ordering — static content first enables KV cache reuse across tickets:
1. **Instructions + conventions** (identical across tickets → cached)
2. **Output format** (identical across tickets → cached)
3. **Task objective** (changes per ticket)
4. **Dynamic context** (changed files, error output — changes every time)

### Agent IO Contracts

All agents MUST follow the **agent-io** skill for input/output formats. Reference by contract name in agent prompts (e.g., "Follow agent-io Contract 1 for output format").

### Model Selection
Choose model by task complexity — use the cheapest model that can do the job well:

| Model | When to use | Cost |
|-------|-------------|------|
| **opus** | Orchestration (main thread), architecture planning, complex reasoning | highest |
| **sonnet** | Code reading, code writing, code review, testing — the workhorse | medium |
| **haiku** | Formatting, doc generation, simple summarization, lightweight reads | lowest (1/3 of sonnet) |

**Rule**: Main thread (orchestrator) should run on **opus**. Launch Claude Code with opus model when using `/ship`.

## Input

$ARGUMENTS

---

## Phase 1: Read Jira Card → `jira-read` skill

Invoke skill **jira-read** with the ticket key from `$ARGUMENTS` or current git branch.

**Gate: WAIT for user confirmation.**

---

## Phase 2: Plan → `planner` agent

### 2.0: Check for existing plan (cross-session state)

Before planning from scratch, check if `/plan-to-jira` already wrote a plan to this Jira ticket:

1. Call `mcp__claude_ai_Atlassian__getJiraIssue` with the ticket key (already fetched in Phase 1)
2. Search the issue's comments for one that starts with `## Implementation Plan`
3. **If found**:
   - Present the existing plan to user:
     ```
     Found an existing Implementation Plan on this ticket (written by /plan-to-jira).

     [show plan content]

     Use this plan? (yes/no)
     - **yes** → skip to Phase 2 step 2 (sanity check) using this plan
     - **no** → proceed to step 1 below and plan from scratch
     ```
   - **Gate: WAIT for user response.**
4. **If not found**: proceed to step 1 below as normal.

**Why**: `/plan-to-jira` writes plans to Jira comments. Without this check, `/ship` would re-plan from scratch and waste time. Jira serves as cross-session persistent memory.

### 2.1: Gather Wiki + Graphify Context (orchestrator does this)

Before dispatching the planner, the orchestrator silently gathers background knowledge:

1. **Personal wiki**: Read `~/wiki/index.md`, find pages related to this ticket's domain. If relevant pages exist, read them and save a short summary.
2. **Project knowledge graph**: If `graphify-out/GRAPH_REPORT.md` exists, read it. If `graphify-out/wiki/index.md` exists, find related community/god node entries.
3. Combine both into a `{wiki_context}` string to pass to the planner. If neither has relevant content, set to "No additional wiki context available."

### 2.2: Plan from scratch (only if no existing plan)

1. Invoke `Agent` with `subagent_type: "planner"`, `model: "opus"`:
   - **Prompt template** (static sections first for KV cache efficiency):
     ```
     ## Instructions
     1. Read CLAUDE.md at $REPO_ROOT/CLAUDE.md for project conventions
     2. Explore relevant codebase areas using Glob/Grep (do NOT wait for file contents to be provided)
     3. Return a step-by-step plan with: exact file paths, phases, risks, dependencies
     4. If anything is unclear, STOP and list your questions — do NOT assume

     ## Output Format
     Return a structured plan with phases, each containing: files to change, what to change, why, risks.

     ## Task
     Plan implementation for: {Jira summary from Phase 1}

     ## Context
     - Jira ticket: {ticket key}
     - Description: {Jira description — short summary, not full dump}

     ## Background Knowledge (from wiki + graphify)
     {wiki_context}
     ```

2. **Sanity check** (orchestrator does this before presenting to user — same as before):
   - Extract all file paths mentioned in the plan
   - Run `ls` or `Glob` to verify each path exists
   - If any path does NOT exist, flag it in the presentation: `⚠️ Plan references {path} but this file does not exist — planner may have hallucinated or the file was renamed`
   - This prevents cascading failures where the entire implementation is based on a wrong file path

3. Present plan to user (including any sanity check warnings)

**Gate: WAIT for user to approve plan.**

---

## Phase 2.5: Create Branch + Worktree for Implementation

After the plan is approved, create the feature branch and an isolated git worktree so that all implementation work happens separately without touching the user's main working tree.

**Why**: The worktree lets agents work in isolation while the user continues on `master`. After implementation, changes are brought back to the main repo on the feature branch so the user can review and commit in batches.

### Steps:

1. **Derive branch name from Jira ticket key**: use the ticket key verbatim as the branch name — `{{JIRA_PROJECT_KEY}}-{TicketNo}` (e.g., `{{JIRA_PROJECT_KEY}}-17959`). **Do NOT append any `-short-description` suffix** — this is the team convention and non-negotiable. The user will reject any name that adds a descriptive suffix.
2. **Present branch name to user for confirmation**:
   ```
   Branch name: {{JIRA_PROJECT_KEY}}-{TicketNo}

   OK? (yes / suggest alternative)
   ```
   **Gate: WAIT for user confirmation on branch name.**

3. **Create worktree with the branch**:
   ```bash
   git worktree add $WORKTREE_ROOT/{{JIRA_PROJECT_KEY}}-{TicketNo} -b {{JIRA_PROJECT_KEY}}-{TicketNo}
   ```
4. **Verify worktree**:
   ```bash
   cd $WORKTREE_ROOT/{{JIRA_PROJECT_KEY}}-{TicketNo} && git status
   ```
5. **Set worktree path variable** — all subsequent phases use this path:
   ```
   WORKTREE=$WORKTREE_ROOT/{{JIRA_PROJECT_KEY}}-{TicketNo}
   BRANCH={{JIRA_PROJECT_KEY}}-{TicketNo}
   ```

**From this point forward, ALL agent prompts must include:**
- `Working directory: {WORKTREE}` — agents MUST `cd` to this directory before any file operations
- `CLAUDE.md path: {WORKTREE}/CLAUDE.md` — conventions file is in the worktree too

**Important**:
- The worktree shares the same git history as the main repo
- Phase 6 brings changes back to the main repo — NO commits happen inside the worktree
- After changes are brought back, the worktree is cleaned up

---

## Phase 3: Implement → multiple agents

> **All Phase 3 agents work in `{WORKTREE}`**, not the main repo directory.

### 3.1: Execute Implementation

Dispatch a single write agent to implement the approved plan:

- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file operations happen here, not the main repo
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md for project conventions
  3. Implement each step of the plan exactly as specified
  4. If any Swagger annotations were added or modified, run `go-swag fmt <directory>` on each affected directory
  5. **Mock regeneration**: If any domain/registry interfaces were added or modified, run `cd {WORKTREE} && go generate` to regenerate mocks via Docker mockery. NEVER write mock files by hand — they are auto-generated.
  6. Do NOT write tests — tests will be handled separately in a later step
  7. Run `cd {WORKTREE} && go build ./...` after implementation to verify it compiles

  ## Output Format
  Return: list of files modified/created, and `build: PASS | FAIL`

  ## Task
  Implement approved plan for Jira ticket {ticket key}: {one-line summary}

  ## Working Directory
  {WORKTREE}

  ## Plan
  {paste the approved plan from Phase 2}
  ```

Wait for this agent to complete before proceeding. Do NOT dispatch any other write agent in parallel — parallel writes cause merge conflicts and race conditions on the same files.

### 3.2: Build Verification
```bash
cd {WORKTREE} && go build ./...
cd {WORKTREE} && go vet ./...
```
If build fails → invoke `Agent` with `subagent_type: "go-build-resolver"`, `model: "sonnet"`:
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file operations happen here
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md
  3. Fix ONLY the build errors — minimal changes, no refactoring
  4. Run `cd {WORKTREE} && go build ./...` and `go vet ./...` after fixes to verify

  ## Output Format
  Follow **Contract 4: Build Fix Result** from Agent IO Contracts section.

  ## Task
  Fix build/vet errors in Go project.

  ## Error Output
  {paste build error output here}
  ```

### 3.3–3.5: Quality Gates (parallel read, then sequential write)

**Step A: Prepare context** (orchestrator does this)
1. Run `cd {WORKTREE} && git diff --name-only` to get changed file list
2. Prepare one-line task summary from Phase 1 Jira card

**Step B: Parallel READ-ONLY analysis** — dispatch in ONE message:

**Agent 1: Code Review** (read-only analysis)
- `subagent_type`: `"go-reviewer"`
- `model`: `"sonnet"`
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file reads happen here
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md for conventions
  3. Read each changed file using the Read tool
  4. Follow `go-reviewer` patterns
  5. DO NOT modify any files — analysis only, because write agents run separately after you

  ## Output Format
  Follow **Contract 1: Code Review Result** from Agent IO Contracts section.

  ## Task
  Review Go code changes for Jira ticket {ticket key}: {one-line summary}

  ## Working Directory
  {WORKTREE}

  ## Changed Files
  {git diff --name-only output, one path per line — paths relative to WORKTREE}
  ```

**Agent 2: Security Review** (read-only analysis, if applicable)
- Only dispatch if changes involve user input, auth, API endpoints, or sensitive data
- `subagent_type`: `"go-security-reviewer"`
- `model`: `"sonnet"`
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file reads happen here
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md
  3. Read each changed file using the Read tool
  4. Focus on: input validation, SQL injection, auth bypass, secrets exposure
  5. DO NOT modify any files — analysis only, because write agents run separately after you

  ## Output Format
  Follow **Contract 2: Security Review Result** from Agent IO Contracts section.

  ## Task
  Security review for Jira ticket {ticket key}: {one-line summary}

  ## Working Directory
  {WORKTREE}

  ## Changed Files
  {git diff --name-only output, one path per line — paths relative to WORKTREE}
  ```

**Step C: Apply fixes sequentially** (only ONE agent writes at a time)

After both read-only agents return, the orchestrator reviews their findings and dispatches a SINGLE write agent to fix issues:

**Agent 3: Apply Code Review Fixes** (write agent — runs alone)
- `subagent_type`: `"go-reviewer"`
- `model`: `"sonnet"`
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file operations happen here
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md
  3. Fix all CRITICAL and HIGH issues listed below
  4. Fix MEDIUM issues when the fix is straightforward
  5. Follow Go conventions from CLAUDE.md
  6. Run `cd {WORKTREE} && go build ./...` after fixes to verify no regressions

  ## Output Format
  Follow **Contract 1: Code Review Result** from Agent IO Contracts section (same format, but after fixes applied).

  ## Task
  Fix the following code review issues in {ticket key}:

  ## Working Directory
  {WORKTREE}

  ## Issues to Fix
  {paste CRITICAL and HIGH issues from Agent 1 results}

  ## Changed Files
  {git diff --name-only output}
  ```

**Step D: Write Tests** (write agent — runs alone, AFTER Step C)

**Agent 4: Write Tests**
- `subagent_type`: `"general-purpose"`
- `model`: `"sonnet"`
- **Prompt template** (static first, dynamic last):
  ```
  ## Instructions
  1. `cd {WORKTREE}` — ALL file operations happen here
  2. Read CLAUDE.md at {WORKTREE}/CLAUDE.md
  3. Follow Go testing conventions from CLAUDE.md (testify, table-driven, `//go:build unit`) patterns
  4. Build tag: `//go:build unit`
  5. Use blank lines to visually separate setup/action/assertion sections — NO AAA comments, because team convention treats them as noise
  6. Use testify for assertions, table-driven tests
  7. **NEVER write mock files by hand**. Mocks are auto-generated by mockery via Docker. If mocks are missing or outdated, run `cd {WORKTREE} && go generate` to regenerate them. If a new interface needs mocks, first check if it's listed in `{WORKTREE}/.mockery.yaml` — if not, add the interface there, then run `go generate`.
  8. Target 80%+ coverage on changed packages
  9. Run: `cd {WORKTREE} && go test -v -tags unit ./path/to/changed/packages/...`

  ## Output Format
  Follow **Contract 3: Test Result** from Agent IO Contracts section.

  ## Task
  Write unit tests for changes in {ticket key}: {one-line summary}

  ## Changed Files
  {git diff --name-only output}
  ```

### 3.5: Integration Tests (after unit tests pass)

Invoke skill **integration-test** after unit tests complete:

```
Follow the integration-test skill at ~/.claude/skills/integration-test.md exactly.

Context:
- Changed files: {git diff --name-only output}
- Ticket: {ticket key}
- Server: localhost:8080 (air, already running or start it)
- Test strategy: Modified existing API → Regression + new parameter tests

Steps to follow from the skill:
1. Step 0: Detect/manage server (auto)
2. Step 1: Determine strategy (Modified API → regression focus)
3. Step 2: Identify all affected endpoints from changed files
4. Step 3: Design test cases (validation errors + happy path + edge cases + regression)
5. Step 4: Execute all test cases via curl
6. Step 5: Present results (Layer 1 summary + Layer 2 detail)
7. Step 6: Export Postman collection to ~/Downloads/postman/
8. Step 7: Prepare Jira summary block (Layer 1 only — saved for Phase 4)
9. Step 8: Cleanup (only kill server if we started it)
```

Save Layer 1 summary output for Phase 4 doc generation.

### 3.6: Present Results

Orchestrator consolidates all agent results into **Contract 5: Quality Gates Summary** format. Display to user AND save for Phase 4 agents.

**Rollback trigger**: If the quality gates show **CRITICAL issues > 3**, the implementation direction may be fundamentally wrong. Instead of hard-fixing, present to user:

```
⚠️ Quality gates found {N} CRITICAL issues. This may indicate the plan needs revision rather than patching.

Options:
1. **Re-plan** — go back to Phase 2 and revise the approach
2. **Fix forward** — attempt to fix all issues in the current approach
3. **Abort** — stop and review manually

Which option? (1/2/3)
```

If user chooses 1 → discard implementation changes (`git checkout .`) and restart from Phase 2.
If user chooses 2 → proceed with fixes as normal.
If user chooses 3 → stop the workflow.

**Gate: WAIT for user confirmation.**

---

## Phase 4: Format Jira Description

### 4.0: Detect marker line — human-confirmed spec boundary

The Jira ticket description (usually written by `/plan-to-jira`) contains a marker line like:
- `以上為人工確認區（PM / 工程師閱讀）。以下為 Agent 實作規格，確認上方內容後交由 Agent 執行。`
- Or similar marker delimiting "human-confirmed spec" (top) from "agent implementation spec" (bottom).

**HARD RULES** (non-negotiable — team convention; the user has rejected violations before):

1. **Everything ABOVE the marker is the human-confirmed spec. Preserve VERBATIM — not a single character may change.** That includes punctuation, whitespace, emoji, existing heading text (`### 概述 / ### 需求 / ### API 變更文件 / ### 已完成的改動`), typos — nothing. The only allowed wrap is nesting the whole top portion under a single `## 目標` H2 (see 4.3).
2. **Everything from the marker line downward is DELETED** (including the marker line itself) and replaced with the template sections below.
3. If no marker exists, the entire current description counts as "above the marker" and is preserved verbatim; new sections are appended below.

Save to variables:
- `description_top` = full content above the marker, VERBATIM
- `write_mode` = `"replace_below_marker"` | `"append"`

### 4.1: Jira Description Template

The following H2 section ordering is the team convention (omit only sections with no content). If your team uses a different format, override this template by editing this section.

- `## 目標`
- `## 邏輯規則`
- `## 實作` (with H3 sub-sections per architectural layer: Service / Flow / API / DB / Config / etc.)
- `## 驗收標準`

**NEVER invent non-template H2 headings** like `## Post-Implementation Summary`, `## 實作變更`, `## 驗收結果`, `## Summary`. Use only the template's H2 names.

### 4.2: Assemble the Description (orchestrator, no agents)

This is a deterministic formatting task — no LLM agents needed. The orchestrator assembles the description directly from Phase 3 output.

Final description structure — exactly this order:

````markdown
## 目標

{description_top — VERBATIM, byte-for-byte preserved. The ONLY allowed transformation: if `description_top` begins with a top-level H2 like `## Implementation Plan`, demote that single line to H3 `### Implementation Plan` so the whole top portion nests as sub-content of `## 目標`. Every other character — punctuation, whitespace, emoji, existing `### 概述 / ### 需求 / ### API 變更文件 / ### 已完成的改動` heading text, tables, code blocks, typos — is copied unchanged.}

---

## 邏輯規則

- {business rule 1 — e.g., "MemberCarrier 以 UID 1:1 查詢，not found 回 (nil, nil)"}
- {rule 2 — cross-brand / null / default-value / edge-case behavior}
- {rule 3 — e.g., "Service X response 共用 GetMember 查詢，但 InfoData 無 Carrier serializer 欄位"}

## 實作

### {Layer 1 — e.g. Service}
- `{file}`: one-line description of what was added

### {Layer 2 — e.g. Flow}
- `{file}`: one-line

### {Layer 3 — e.g. Serializer / API / Delivery}
- `{file}`: one-line

### DB / Config
- {schema change, or "無 schema 變更（entity + migration 已於 {{JIRA_PROJECT_KEY}}-XXXXX 完成）"}

## 驗收標準

- [x] {endpoint / behavior verified} — {observed outcome}
- [x] Regression: {existing fields unaffected}
- [x] `go build ./...` / `go vet` / `gofmt` 通過
- [x] Unit test `{TestName}` N/N PASS
- 測試環境：{localhost (air + dev VPN DB) | staging | ...}
- PR: [#{num}]({url}) — **only include if PR is already created; otherwise add after Phase 6**
````

**Pre-preview checklist** (orchestrator MUST verify before showing to user):
- [ ] `description_top` section content is byte-for-byte identical to what was above the marker (only allowed diff: one `##` → `###` demotion of the top-level heading)
- [ ] No `## Post-Implementation Summary`, `## 實作變更`, `## 驗收結果`, `## Summary` anywhere
- [ ] H2 order is `## 目標` → `## 邏輯規則` → `## 實作` → `## 驗收標準`
- [ ] `## 實作` H3 subsections group by architecture layer (Service / Flow / API / DB), not by file

### 4.3: Present Preview & Confirm

```
## Jira Description Preview

Structure: ## 目標 (verbatim human spec) → ## 邏輯規則 → ## 實作 → ## 驗收標準
Write mode: {write_mode}
Top portion preserved: yes (byte-equal verified)

---

[Full assembled description content]

---

This will REPLACE {ticket key} description.
Type **ok** to write, **edit** to modify specific sections, or **no** to skip.
```

**Gate: WAIT for user confirmation before writing to Jira.**

---

## Phase 5: Write to Jira → Atlassian MCP

Call `mcp__claude_ai_Atlassian__editJiraIssue`:
- `cloudId`: `{{JIRA_CLOUD_ID}}`
- `issueIdOrKey`: ticket key from Phase 1
- `fields.description`: the assembled description from Phase 4.7 (markdown, ADF-convertible)

**Important**: Only update the `description` field. Do NOT touch summary, status, assignee, or any other field.

Confirm success:
```
Jira description updated.
- **Issue**: [{{JIRA_PROJECT_KEY}}-XXXXX](https://{{JIRA_DOMAIN}}/browse/{{JIRA_PROJECT_KEY}}-XXXXX)
- **Top portion**: preserved (human spec)
- **Bottom portion**: replaced with post-implementation summary
```

---

## Phase 6: Bring Changes Back to Local Repo

After implementation and review are complete, bring all changes from the worktree back to the main repo on the feature branch. **NO commits happen in this phase** — the user will commit manually in batches.

### 6.1: List Changed Files in Worktree
```bash
cd {WORKTREE} && git diff --name-only HEAD
cd {WORKTREE} && git diff --stat HEAD
```
Present the file list to user:
```
Worktree 裡的修改檔案：

{file list with stats}

準備把這些檔案搬回本地 repo 的 {BRANCH} branch。
OK?
```

**Gate: WAIT for user confirmation.**

### 6.2: Switch Main Repo to Feature Branch
```bash
cd $REPO_ROOT && git checkout {BRANCH}
```

### 6.3: Copy Changed Files from Worktree to Main Repo
For each changed/new file, copy from worktree to main repo:
```bash
# For each file in the diff list:
cp {WORKTREE}/{file} $REPO_ROOT/{file}
```
For deleted files, remove them from main repo:
```bash
rm $REPO_ROOT/{deleted_file}
```

### 6.4: Verify Changes Landed Correctly
```bash
cd $REPO_ROOT && git status
cd $REPO_ROOT && go build ./...
```

### 6.5: Cleanup Worktree
```bash
git worktree remove {WORKTREE}
```

### 6.6: Present Result
```
✅ 所有修改已搬回本地 repo，目前在 branch: {BRANCH}

修改檔案：
{file list}

你現在可以用 git add / git commit 分批提交。
建議提交順序：schema → logic → API → validation → tests → config

Worktree 已清除。
```

**After user has committed and pushed**, proceed to Phase 7 if they request PR creation. Otherwise the workflow ends here.

---

## Phase 7: Create PR — only on explicit user request

Trigger: user asks to create a PR, or says "push / PR it / 發 PR / 建 PR".

### 7.1: Verify remote branch exists

```bash
cd $REPO_ROOT && git ls-remote --heads origin {BRANCH}
```
If empty → ask user to push first. Do NOT push for them.

### 7.2: Load the PR template

Read the project's PR template file (if it exists):
```
$REPO_ROOT/.github/pull_request_template.md
```
Save the full content as `pr_template`. If no template file exists, fall back to the default template in section 7.4 below.

**HARD RULES** (non-negotiable — team convention):
- PR body MUST use the section headings from `pr_template` exactly as-is (`## Objective`, `## Description`, `## RFC doc`, `## Root Cause` (bugs only), `## Solution` (bugs only), `## Checklist`).
- Do NOT invent alternative headings like `## Summary`, `## Changes`, `## Response shape`, `## Test plan`, `## Out of scope`.
- Checklist items in the template are fixed — keep every bullet, tick `[x]` only what is actually verified.
- If Issue Type ≠ Bug, delete the `## Root Cause` and `## Solution` headings (template explicitly says so).

### 7.3: Gather inputs

- Jira ticket key + summary (from Phase 1)
- Changed files + diff summary: `git diff --stat master...{BRANCH}`
- Quality-gate results (from Phase 3.6)
- Integration / regression outcomes (from Phase 3.5)

### 7.4: Assemble the PR body

Structure (omit `## Root Cause` / `## Solution` for non-bug issues):

````markdown
## Objective

- [{ticket key}](https://{{JIRA_DOMAIN}}/browse/{ticket key})
- {one-line PR subject}

## Description

{2–5 sentences explaining purpose / behavior / scope of influence. Include any non-obvious detail reviewers need (brand parity, null handling, cross-service impact). Keep technical, no marketing.}

{Optional: short table of the main changes grouped by architectural layer — only if it genuinely helps the reviewer; skip if diff is small.}

## RFC doc

- N/A

## Checklist

{Copy the template's checklist verbatim; tick items that are actually verified}
- [x] Unit tests (...)
- [x] Manual test on local or dev
- [x] Self review
- [x] Confirm compliance with team API guide ({{API_GUIDE_URL}})
- [ ] Other things need to check (e.g. i18n / confirm with FE / etc.)
````

**Pre-preview checklist** (orchestrator MUST verify):
- [ ] H2 headings match the template (no invented ones)
- [ ] Jira link present at the top of `## Objective`
- [ ] `## Root Cause` / `## Solution` only appear if Issue Type == Bug
- [ ] All original checklist bullets from template are present

### 7.5: Present preview & confirm

```
## PR Preview

Title: [{ticket key}] {subject}
Base: master  Head: {BRANCH}
Template compliance: verified (Objective / Description / RFC doc / Checklist)

---

[Full PR body]

---

Type **ok** to create PR, **edit** to modify, or **no** to skip.
```

**Gate: WAIT for user confirmation.**

### 7.6: Create PR

```bash
gh pr create --base master --head {BRANCH} \
  --title "[{ticket key}] {subject}" \
  --body "$(cat <<'EOF'
{assembled PR body}
EOF
)"
```

Report the PR URL to the user.

### 7.7: Update Jira description with PR link (optional)

If Phase 5 already wrote the Jira description with a placeholder for PR link, patch `## 驗收標準` section to include `- PR: [#{num}]({url})`. Use `editJiraIssue` with the previous full description + the PR line appended to `## 驗收標準`. Only update the `description` field.

**Workflow ends here.**

---

## Summary: Skills & Agents

| Phase | Tool | Type | Model | Mode |
|-------|------|------|-------|------|
| — | **main thread (orchestrator)** | — | **opus** | dispatcher |
| 1 | `jira-read` | skill | opus (inherit) | main thread |
| 2 | `planner` | agent | **opus** | subagent |
| 2.5 | branch + worktree | bash | — | main thread |
| 3.1 | implementation | agent | **sonnet** | subagent (write) → |
| 3.2 | `go-build-resolver` | agent (on failure) | **sonnet** | subagent (write) |
| 3.3 | `go-reviewer` | agent | **sonnet** | subagent (read) ∥ |
| 3.4 | `go-security-reviewer` | agent (if applicable) | **sonnet** | subagent (read) ∥ |
| 3.3→fix | `go-reviewer` | agent | **sonnet** | subagent (write) → |
| 3.4 | `go-tdd-guide` | agent | **sonnet** | subagent (write) → |
| 3.5 | `integration-test` (Hurl) | skill | opus (inherit) | main thread → |
| 4.1 | Jira description template (inline) | — | — | main thread |
| 4.2 | assemble description (template H2s) | — | — | main thread |
| 4.3 | preview & confirm | — | — | main thread |
| 5 | `editJiraIssue` (description only) | MCP | — | main thread |
| 6 | bring changes back | bash (cp + checkout) | — | main thread |
| 7.2 | read `.github/pull_request_template.md` | Read | — | main thread |
| 7.6 | `gh pr create` with template-compliant body | bash | — | main thread (only on user request) |

**Legend**: ∥ = parallel (read-only), → = sequential (write, must wait for previous)

**Post-ship (user takes over)**: `git add` / `git commit` 分批提交 → `git push`. If user then asks for a PR, proceed to Phase 7.

## Confirmation Gates

1. Phase 1: After reading Jira card
2. Phase 2: After presenting plan
3. Phase 2.5: After confirming branch name (must be `{{JIRA_PROJECT_KEY}}-{TicketNo}` exactly, no suffix)
4. Phase 3.6: After implementation + review + unit tests + integration tests
5. Phase 4.3: After Jira description preview (BEFORE writing to Jira — verify template H2 order and verbatim preservation of human-confirmed spec)
6. Phase 6.1: Before bringing changes back to local repo
7. Phase 7.5: After PR body preview (only if user requested PR — verify PR template compliance)

