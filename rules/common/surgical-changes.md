# Surgical Changes

Adapted from [Karpathy's LLM coding guidelines](https://x.com/karpathy/status/2015883857489522876). Applies to ALL code changes — inside skills (`/plan-to-jira`, `/ship`) and outside.

## 1. Think Before Coding

Surface assumptions, don't hide confusion:

- State assumptions explicitly. If uncertain, ask via `AskUserQuestion`.
- If multiple interpretations exist, present them — don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, STOP. Name what's confusing. Ask.

## 2. Simplicity First

Minimum code that solves the problem:

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you wrote 200 lines and it could be 50, rewrite it.

Self-check: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes (most important)

Touch only what the task requires:

- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it — don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless explicitly asked.

**Test**: Every changed line must trace directly to the user's request. If a line doesn't, remove it before submitting.

## 4. Goal-Driven Execution

Define verifiable success criteria, then loop until verified:

- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan with verification checks per step. Strong success criteria let you loop independently; weak criteria ("make it work") require constant clarification.

## Interaction with this harness's workflows

- `/plan-to-jira`: these rules reinforce Layer 1/2's "不確定就問,不假設" — same spirit, more specific on code behavior.
- `/ship` Phase 3: applies directly to implementation agent and review-fix agent — "只改計畫列的檔案,不順手重構"。
- Outside skills: applies to all direct code edits.
