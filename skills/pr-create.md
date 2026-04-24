---
name: pr-create
description: Create a PR using the repo's PR template and Jira ticket metadata
---

## PR Creation

Create a Pull Request following the project's PR template and Jira ticket conventions.

### Step 1: Gather Context

1. Run `git diff master...HEAD` to see all changes
2. Run `git log master..HEAD --oneline` to see all commits
3. Check for migration files:
   - `migrations/mysql` → extract GOOSE migration number (remove zero-padding)
   - `migrations/mongodb` → extract DB name and migration number
4. Read `$REPO_ROOT/.github/pull_request_template.md` if it exists — use its exact heading structure

### Step 2: Build PR Title

**Format**:
- Normal: `[{{JIRA_PROJECT_KEY}}-{TicketNo}] {Component} - {Description}`
- Fix PR (branch contains `-FIX{N}`): `[{{JIRA_PROJECT_KEY}}-{TicketNo}-FIX{N}] {Component} - {Description}`

Ticket number is extracted from the branch name.

### Step 3: Build PR Body

Use `$REPO_ROOT/.github/pull_request_template.md` as the base (if it exists). Otherwise use the default template below:

```markdown
## Objective

- [{{JIRA_PROJECT_KEY}}-{TicketNo}](https://{{JIRA_DOMAIN}}/browse/{{JIRA_PROJECT_KEY}}-{TicketNo})
- [GOOSE-{N}] (only if MySQL migration exists, remove zero-padding)
- [MONGO-{DbName}-{N}] (only if MongoDB migration exists)
- {PR subject — same as title without bracket prefix}

## Description

[繁體中文描述]

### 改動範圍
[列出主要改動的模組/檔案及說明]

### API 異動
[若有 API 變更:endpoint table + 新增欄位 table]

### 儲存層異動
[若有 ES/MongoDB/MySQL 變更:簡要說明]

### 流程圖
[Mermaid diagrams if they help reviewers understand the change]

## RFC doc

- N/A (or link if provided by user)

## Root Cause
> Delete this ENTIRE section if NOT a Bug ticket or Fix PR

[Root cause — for Fix PR: find QA test cases from Jira comments + commit content]

## Solution
> Delete this ENTIRE section if NOT a Bug ticket or Fix PR

[Solution description]

## Checklist

Please make sure to complete the following checklist before merge or remove [WIP]
- [x] Unit tests (explain in Description if any reasons why it is difficult to write tests, such as adjusting payment API)
- [x] Manual test on local or dev
- [x] Self review
- [x] Confirm compliance with team API guide ({{API_GUIDE_URL}})
- [ ] Other things need to check (e.g. i18n / confirm with FE / etc.)
```

### Step 4: Present PR Preview

```
## PR Preview

**Title**: [{{JIRA_PROJECT_KEY}}-XXXXX] Component - Description

[Full PR body]

---

Type **ok** to create PR, **edit** to modify, or **no** to cancel.
```

**WAIT for user confirmation.**

### Step 5: Create PR

```bash
gh pr create --title "[title]" --body "[body]" --base master
```

Do NOT use `--draft`. Create a normal PR.

### Step 6: Confirm

```
PR created successfully!
- **URL**: [PR URL]
- **Base**: master
- **Head**: [branch name]
```

### Rules

1. **Prefer the repo's own template** (`$REPO_ROOT/.github/pull_request_template.md`) over the fallback above when one exists
2. **Delete Root Cause / Solution sections entirely** if not a Bug or Fix PR
3. **Migration markers**: only include if migration files exist in the diff
4. **Description in 繁體中文**, technical terms in English
5. **RFC doc**: ask the user if they have a link; if not, fill `N/A`
6. **Fix PR special handling**: use commits content + Jira comments for Root Cause, not the full ticket description
7. **WAIT for user confirmation** before creating the PR
