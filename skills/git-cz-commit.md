---
name: git-cz-commit
description: Stage and commit changes using git cz (commitizen) conventional commit format with logical grouping
---

## Git Cz Style Commits

Stage and commit changes following the **commitizen (git cz)** conventional commit format, grouped logically by concern.

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

### Fields

| Field | Required | Rules |
|-------|----------|-------|
| **type** | Yes | One of: `feat`, `fix`, `refactor`, `test`, `chore`, `perf`, `docs`, `ci`, `style` |
| **scope** | Yes | Affected module/package. Examples: `review`, `search`, `es`, `order`, `validator` |
| **subject** | Yes | Imperative mood, lowercase, no period at end. Max 72 chars |
| **body** | Optional | Explain WHAT and WHY (not HOW). Wrap at 100 chars. Blank line between subject and body |
| **footer** | Optional | `BREAKING CHANGE: ...` or `Refs: {{JIRA_PROJECT_KEY}}-XXXXX` |

### Type Reference

| Type | When to Use |
|------|------------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code restructure without behavior change |
| `test` | Adding or updating tests |
| `chore` | Build, config, tooling changes |
| `perf` | Performance improvement |
| `docs` | Documentation only |
| `ci` | CI/CD pipeline changes |
| `style` | Formatting, whitespace (no logic change) |

### Grouping Strategy

Analyze all changed files and group into separate commits by concern, in this order:

1. **Schema/Storage** (ES mapping, DB migration, MongoDB schema):
   ```
   feat(es): add sell_time_hhmm keyword field to sell_product index
   ```

2. **Core business logic** (services, domain, usecase):
   ```
   feat(search): implement filter_times time slot filtering with ±30min range
   ```

3. **API layer** (handlers, routes, request/response structs):
   ```
   feat(api): support filter_times[] query parameter in branch search endpoints
   ```

4. **Validation / Error handling**:
   ```
   feat(validator): add hh:mm format validation for filter_times parameter
   ```

5. **Tests**:
   ```
   test(search): add unit tests for time slot parsing and boundary clamping
   ```

6. **Config / Infra / Other**:
   ```
   chore(config): update ES mapping template for sell_time_hhmm
   ```

### Execution

For each logical group:

```bash
git add [specific files only]
git commit -m "$(cat <<'COMMIT'
<type>(<scope>): <subject>

<body if needed>

Refs: {{JIRA_PROJECT_KEY}}-{TicketNo}
COMMIT
)"
```

### Rules

1. **Stage specific files only** — NEVER use `git add -A` or `git add .`
2. **Each commit must be independently meaningful** — could be cherry-picked alone
3. **Order**: schema → business logic → API → validation → tests → config
4. **One concern per commit** — don't mix test files with business logic
5. **Subject line max 72 chars** — use body for details
6. **Footer always includes** `Refs: {{JIRA_PROJECT_KEY}}-{TicketNo}` with the current ticket number
7. **If a commit has related files across layers** (e.g., a new entity + its repository), group them together if they serve the same purpose
