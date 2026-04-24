---
name: jira-read
description: Read a Jira ticket and present structured requirements to the user
---

## Jira Card Reader

Read a Jira ticket and present its content in a structured format for implementation.

### Constants

- **Cloud ID**: `{{JIRA_CLOUD_ID}}`
- **Content Format**: `markdown`

### Step 1: Resolve Ticket Key

1. If a ticket key is provided (matches `{{JIRA_PROJECT_KEY}}-\d+`), use it directly
2. Otherwise, run `git branch --show-current` and extract `{{JIRA_PROJECT_KEY}}-\d+` from the branch name
3. If neither works, ask the user for the ticket key

### Step 2: Read Jira Ticket

Call `mcp__claude_ai_Atlassian__getJiraIssue`:
- `cloudId`: `{{JIRA_CLOUD_ID}}`
- `issueIdOrKey`: extracted ticket key
- `responseContentFormat`: `markdown`

Extract and store:
- **Summary** (title)
- **Description** (implementation details)
- **Issue Type** (Bug / Task / Sub-task / Story)
- **Status**
- **Parent** (if sub-task, read parent summary too)

### Step 3: Present Requirements

Display to the user:

```
## Jira Card: {{JIRA_PROJECT_KEY}}-XXXXX
**Type**: [Bug/Task/...]
**Summary**: [title]
**Description**: [content]
**Parent**: [parent ticket if exists]

Ready to start implementation? (yes / need clarification)
```

**WAIT for user confirmation before proceeding.**

### Output

Return the following data for use by subsequent skills:
- Ticket key (e.g. `{{JIRA_PROJECT_KEY}}-17669`)
- Issue type (Bug / Task / Sub-task / Story)
- Summary
- Description
- Parent summary (if exists)
