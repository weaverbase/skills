---
name: executing-plans-with-subagents
description: Use when executing an approved multi-task implementation plan by dispatching implementer and reviewer subagents; when deciding whether to act on reviewer findings; when a review/fix loop keeps expanding; or when a subagent edits outside its task.
---

# Executing Plans with Subagents

This skill is self-contained. Do not load other code-review or TDD skills as implicit dependencies.

## Mission

Deliver the approved feature accurately and safely. The goal is **verified acceptance**, not eliminating every possible concern or improving the entire codebase.

**Non-negotiable:** confirmed security/data-loss flaws, unmet acceptance criteria, and regressions introduced by this change cannot be silently deferred. If they cannot be resolved within the budget, stop the affected task and report the blocker; do not mark it done.

## Roles and authority

- **Coordinator (you):** owns scope, triage, budget, decisions, task ledger, and final delivery. Only the coordinator can authorize a fix, expansion of scope, or change to the plan.
- **Implementer subagent:** implements exactly one task, runs focused verification and self-checks; does not spawn reviewers or other subagents.
- **Reviewer subagent:** independently reviews **only the task's baseline-to-current diff**, the task brief, and relevant immediate context. Read-only; reports findings and evidence; never edits code, dispatches agents, or creates new requirements.
- **Fixer:** preferably the same implementer if resumable; makes **only coordinator-approved corrections**. No opportunistic cleanup.

If independent subagents are unavailable, state that limitation and execute the same roles sequentially without claiming independent review.

## Inputs / prerequisites

Use when there is an approved plan with ordered tasks and explicit acceptance criteria. If missing, first write a minimal plan with acceptance criteria and obtain approval where required by the user's instruction. Do not treat reviewer suggestions as changes to an approved plan.

For each task capture:
- Task ID and objective
- Acceptance criteria and relevant interfaces
- Explicit **out of scope** notes
- File or component area and dependency context
- Baseline commit / working-tree state; preserve pre-existing user changes
- Minimal verification commands that exist in the repository

Never overwrite or revert unrelated user changes. Committing is outside this skill; leave it to the user or their workflow.

## Review budget

Per task: **one implementer, one read-only reviewer, at most one fix wave, one targeted re-review.** A blocker still unresolved after that → `BLOCKED` and escalate, **never accept**. Only the user can authorize another bounded wave. Details and templates: [references/triage-and-fix.md](references/triage-and-fix.md).

## Execution procedure

### 0. Preflight

1. Read the approved plan and inspect repo state (`git status`, where appropriate).
2. Snapshot a baseline before each task. A temporary index leaves the working tree and the user's staging untouched:
   ```sh
   snap() { GIT_INDEX_FILE=$(mktemp -u) sh -c 'git read-tree HEAD && git add -A && git write-tree; rm -f "$GIT_INDEX_FILE"'; }
   BASE=$(snap)                 # before the task
   git diff "$BASE" "$(snap)"   # after: this task's changes only, new files included
   ```
3. Identify task's acceptance criteria, in-scope files, and mandatory verification.
4. Create an in-session ledger: `PENDING | IMPLEMENTING | REVIEWING | FIXING | DONE | BLOCKED` with task IDs and deferred observations.
5. Do not undertake general refactors, test harness construction, framework migration, or unrelated failing-test repair without explicit authorization.

### 1. Dispatch implementer

Read [implementer-prompt.md](implementer-prompt.md). Supply **the exact task brief**, relevant architectural constraints, the expected result, out-of-scope constraints, and the intended verification. Keep context concise. One implementer writes at a time; avoid concurrent edits to a shared worktree.

On report:
- `DONE`: proceed to review.
- `NEEDS_CONTEXT`: answer essential questions before implementation resumes.
- `BLOCKED`: identify the precise blocker; don't silently pivot into infrastructure work.
- `DONE_WITH_CONCERNS`: triage concerns before review; do not automatically enlarge scope.

### 2. Construct bounded review package

Include only:
- Task brief and acceptance criteria
- Task-only diff: `git diff "$BASE" "$(snap)"`
- Changed-file list and key interfaces
- Implementer's test command/results and known limitations

If task changes are not separable from existing work, disclose that and review manually scoped changed files/hunks; do not imply perfect isolation.

### 3. Dispatch exactly one task reviewer

Read [task-reviewer-prompt.md](task-reviewer-prompt.md). Make it **read-only** if tooling permits. Reviewer returns `PASS`, `PASS_WITH_NOTES`, or `BLOCK`, with evidence for every `BLOCKER`. No reviewer-written patches.

Reviewer may report a necessary critical issue discovered nearby, but it must show a concrete affected path or exploit/data-loss scenario. Broad architecture/performance speculation is not a blocker.

### 4. Coordinator triage (mandatory; never auto-fix a review list)

Classify every finding as `BLOCKER`, `UNKNOWN`, or `DEFER` using [references/triage-and-fix.md](references/triage-and-fix.md). Dispatch the fixer only for approved `BLOCKER`s, with exact file/line, reproduction, expected result, and boundaries.

### 5. One fix wave → targeted re-review

Use the fix-wave and re-review templates in [references/triage-and-fix.md](references/triage-and-fix.md); **do not reopen a broad quality hunt**.

If resolved → `DONE`. If not → `BLOCKED` with a concise obstacle and smallest sensible next step. Do not silently proceed past a blocking dependency; independent unaffected tasks may continue if safe.

### 6. Final feature verification and delivery

Read [references/final-verification.md](references/final-verification.md). Verify behavior across tasks once, using existing tools/tests. If a broader test suite fails, classify the failure by evidence rather than rewriting the test framework. Do not declare full success with unresolved required checks.

Report: implemented scope; acceptance criteria; tests passed/failed/not run; remaining blockers; deferred observations (limited to actionable items). Never claim tests passed without results.

## Completion criteria

Task `DONE` requires all of:
1. Its acceptance criteria met.
2. Focused verification passing, or an explicit justified alternative if not executable; mandatory verification failures remain blocking.
3. No unresolved `BLOCKER`s.
4. No unapproved scope expansion.

Feature `DONE` additionally requires integration of tasks and required feature-level verification.

## Hard stop rules

- Reviewer creates a new requirement or recommends a framework migration → defer and return to approved scope.
- New failure appears → first classify *introduced / pre-existing / environment / unknown* with minimal diagnosis.
- Reviewer demands more edge-case tests without a concrete failure mode material to the task → defer.
- A required criterion cannot be verified with existing tests/tools → mark `BLOCKED` and ask the user to accept it unverified or authorize the tooling. Never build a harness on your own judgment.
- Any subagent edits outside its assigned task → stop and inspect `git diff "$BASE" "$(snap)"`. Revert only hunks that appear in that diff and fall outside the task's files; the snapshot cannot tell them apart from user edits made during the task, so if unsure, ask the user before reverting.

## Related files

- [implementer-prompt.md](implementer-prompt.md)
- [task-reviewer-prompt.md](task-reviewer-prompt.md)
- [references/triage-and-fix.md](references/triage-and-fix.md)
- [references/final-verification.md](references/final-verification.md)
