# Agent IO Contracts

Standardized input/output formats for agent communication in `/ship` workflow.
All agents MUST follow these contracts so orchestrator and downstream consumers can reliably parse results.

---

## Contract 1: Code Review Result

**Producer**: go-reviewer agent (read-only)
**Consumer**: orchestrator -> go-reviewer agent (write fix)

```
### Issues
- CRITICAL: {description} | {file}:{line}
- HIGH: {description} | {file}:{line}
- MEDIUM: {description} | {file}:{line}
### Summary
- total: {n}, critical: {n}, high: {n}, medium: {n}
- status: PASS | FAIL
```

- One issue per line, `|` separator between description and location
- FAIL if any CRITICAL or HIGH
- Orchestrator passes only CRITICAL+HIGH lines to write-fix agent

---

## Contract 2: Security Review Result

**Producer**: security-reviewer agent (read-only)
**Consumer**: orchestrator

```
### Vulnerabilities
- CRITICAL: {description} | {file}:{line} | {category}
- HIGH: {description} | {file}:{line} | {category}
### Summary
- total: {n}, status: PASS | FAIL
```

- Category: injection, auth, xss, secrets, csrf, other
- PASS if zero CRITICAL and zero HIGH

---

## Contract 3: Test Result

**Producer**: tdd-guide agent (write)
**Consumer**: orchestrator

```
### Files Created
- {test_file_path}
### Summary
- test_count: {n}, coverage: {n}%, status: PASS | FAIL
```

- PASS if all tests pass AND coverage >= 80%

---

## Contract 4: Build Fix Result

**Producer**: go-build-resolver agent (write)
**Consumer**: orchestrator

```
### Files Modified
- {file}: {what was fixed}
### Summary
- fixes: {n}, build: PASS | FAIL, vet: PASS | FAIL
```

---

## Contract 5: Quality Gates Summary

**Producer**: orchestrator (Phase 3.6, assembled from Contract 1-4)
**Consumer**: haiku doc agents (Phase 4.2-4.5)

```
- build: PASS | FAIL
- vet: PASS | FAIL
- review: {critical} critical, {high} high, {medium} medium -> {fixed} fixed
- security: PASS | FAIL | N/A
- tests: {count} tests, {coverage}% coverage
- changed_files: {comma-separated paths}
- ticket: {{{JIRA_PROJECT_KEY}}-XXXXX}
- summary: {one-line description}
```

- Each line is key: value, parseable by agents
- Passed verbatim as shared context to all Phase 4 agents

---

## Contract 6: Doc Section Output

**Producer**: haiku doc agents (Phase 4.2-4.5)
**Consumer**: orchestrator (Phase 4.6, concatenation)

Each agent returns markdown with H3 headings:

- Agent A: `### Summary` or `### Bug Report`
- Agent B: `### API Changes`
- Agent C: `### Storage Changes`
- Agent D: `### Diagrams` and `### Testing Notes`

Rules:
- Use H3 headings, orchestrator wraps in H2
- Mermaid in fenced mermaid code blocks
- Tables in pipe syntax
- If not applicable: `### {Title}` followed by N/A
- No frontmatter, no metadata

---

## How to Reference

In agent prompts:
```
"Follow agent-io Contract {N} for output format."
```
