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

- **Coordinator (you):** owns scope, triage, budget, decisions, persistent task ledger (sole writer), and final delivery. Only the coordinator can authorize a fix, expansion of scope, or change to the plan, within user authorization; only the user can extend the fix-wave budget.
- **Implementer subagent:** implements exactly one task, runs focused verification and self-checks; does not spawn reviewers or other subagents.
- **Reviewer subagent:** independently reviews **only the task's baseline-to-current diff**, the task brief, and relevant immediate context. Read-only; reports findings and evidence; never edits code, dispatches agents, or creates new requirements.
- **Fixer:** preferably the same implementer if resumable; makes **only coordinator-approved corrections**. No opportunistic cleanup.

If independent subagents are unavailable, state that limitation and execute the same roles sequentially without claiming independent review or marking its completion requirement met.

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

1. Resolve the target repository root and approved repository-relative plan path; read the plan and inspect Git status/history and scoped implementation evidence. Locate/init/reconcile `.agent-work/executing-plans-with-subagents/<plan-id>/progress.md` using [references/progress-ledger.md](references/progress-ledger.md). Run its resume/revision protocol on every new session and context recovery before selecting an eligible task; skip still-valid completed work.
2. For each **new** task, snapshot and persist its original baseline before dispatch. On recovery reuse it; never capture partial implementation as a replacement baseline. Run this example from the resolved target repository root. It requires an existing `HEAD` commit (not unborn-repository support). A temporary index leaves the working tree and the user's staging untouched; both snapshots exclude the entire runtime subtree, even previously tracked files:
   ```sh
   REPO_ROOT=$(git rev-parse --show-toplevel) || exit 1
   snap() (
     cd "$REPO_ROOT" || exit 1
     snap_dir=$(mktemp -d) || exit 1
     export GIT_INDEX_FILE="$snap_dir/index"
     trap 'rm -f "$GIT_INDEX_FILE" "$GIT_INDEX_FILE.lock"; rmdir "$snap_dir"' 0
     trap 'exit 1' HUP INT TERM
     git read-tree HEAD || exit 1
     git add -A -- . ':(top,exclude,literal).agent-work/executing-plans-with-subagents' || exit 1
     git rm -r -f --cached --ignore-unmatch -- ':(top,literal).agent-work/executing-plans-with-subagents' >/dev/null || exit 1
     git write-tree || exit 1
   )
   BASE=$(snap) || exit 1       # original baseline, new task only; persist before dispatch
   ```
3. Identify task's acceptance criteria, in-scope files, and mandatory verification.
4. Persist task status, separate active phase, per-task evidence, review stage, next action, and durable fix budget per the ledger reference. Stop on malformed/contradictory state or failed writes. Never overwrite user work to reconcile it; missing baseline evidence may block required review.
5. Do not undertake general refactors, test harness construction, framework migration, or unrelated failing-test repair without explicit authorization.

### 1. Dispatch implementer

Read [implementer-prompt.md](implementer-prompt.md). Supply **the exact task brief**, relevant architectural constraints, the expected result, out-of-scope constraints, and the intended verification. Keep context concise. One implementer writes at a time; avoid concurrent edits to a shared worktree.

Before initial dispatch, persist original baseline, task `IN_PROGRESS`, phase `IMPLEMENTING`, and initial review/budget state. Pass the ledger/runtime subtree as out of scope; implementers report evidence, never update it. After interruption, resume only unfinished work within the saved state.

On report:
- `DONE`: persist files, result fingerprint and verification evidence, then phase `REVIEWING` / initial before review; this is not ledger `DONE`.
- `NEEDS_CONTEXT`: answer essential questions before implementation resumes.
- `BLOCKED`: identify the precise blocker; don't silently pivot into infrastructure work.
- `DONE_WITH_CONCERNS`: triage concerns before review; do not automatically enlarge scope.

### 2. Construct bounded review package

Include only:
- Task brief and acceptance criteria
- Task-only diff against the saved original baseline (new files included; runtime subtree excluded):
  ```sh
  AFTER=$(snap) || exit 1
  git diff "$BASE" "$AFTER" -- || exit 1
  ```
- Changed-file list and key interfaces
- Implementer's test command/results and known limitations

If task changes are not separable from existing work, disclose that and review manually scoped changed files/hunks; do not imply perfect isolation.

### 3. Dispatch exactly one task reviewer

Read [task-reviewer-prompt.md](task-reviewer-prompt.md). Make it **read-only** if tooling permits. Reviewer returns `PASS`, `PASS_WITH_NOTES`, or `BLOCK`, with evidence for every `BLOCKER`. No reviewer-written patches.

Reviewer may report a necessary critical issue discovered nearby, but it must show a concrete affected path or exploit/data-loss scenario. Broad architecture/performance speculation is not a blocker.

Persist review completion, checked state, and initial versus targeted stage before advancing. On recovery reuse valid completed reviews; do not repeat a broad initial review after fixes.

### 4. Coordinator triage (mandatory; never auto-fix a review list)

Classify every finding as `BLOCKER`, `UNKNOWN`, or `DEFER` using [references/triage-and-fix.md](references/triage-and-fix.md). Dispatch the fixer only for approved `BLOCKER`s, with exact file/line, reproduction, expected result, and boundaries.

Persist the review outcome, task-keyed decisions and deferred evidence before any fix dispatch. Map `PASS_WITH_NOTES` to ledger `PASS_WITH_DEFERRED` only after nonblocking triage; `BLOCK` is evidence, not an automatic fix list.

### 5. One fix wave → targeted re-review

Use the fix-wave and re-review templates in [references/triage-and-fix.md](references/triage-and-fix.md); **do not reopen a broad quality hunt**.

Before dispatch, persist authorized blockers/scope, separate pre-fix baseline, phase `FIXING`, and increment `Fix Waves Used` to reserve the wave. Recovery continues only unfinished corrections in that same wave, never resets the budget. Persist fix results, phase `VERIFYING` and pending checks, then their results and phase `REVIEWING` / targeted before the one targeted re-review.

If completion criteria below are met, persist `DONE`. Otherwise persist `BLOCKED` with evidence, consumed budget, a concise obstacle and smallest sensible next step. Only explicit user authorization can add a newly finite wave budget. Do not silently proceed past a blocking dependency; independent unaffected tasks may continue if safe.

### 6. Final feature verification and delivery

Read [references/final-verification.md](references/final-verification.md). Verify behavior across tasks once, using existing tools/tests. If a broader test suite fails, classify the failure by evidence rather than rewriting the test framework. Do not declare full success with unresolved required checks.

Reconcile required tasks and saved feature-verification evidence first; resume outstanding checks, not still-valid completed ones. Persist feature checked state, commands/results, pending actions, applicable budget and overall status before delivery. This checkpoint adds no fresh fix allowance or mandatory final reviewer.

Report: implemented scope; acceptance criteria; tests passed/failed/not run; remaining blockers; deferred observations (limited to actionable items). Never claim tests passed without results.

## Completion criteria

Task `DONE` requires all of:
1. Its acceptance criteria met.
2. Focused verification passing, or an explicit justified alternative that establishes acceptance if not executable; merely documenting an unavailable check is insufficient, and mandatory verification failures remain blocking.
3. Independent task review completed with no required acceptance left `UNVERIFIED`, including targeted re-review after a fix wave.
4. No unresolved `BLOCKER`s.
5. No unapproved scope expansion; evidence and final task state persisted.

Feature `DONE` additionally requires integration of tasks and required feature-level verification.

## Hard stop rules

- Reviewer creates a new requirement or recommends a framework migration → defer and return to approved scope.
- New failure appears → first classify *introduced / pre-existing / environment / unknown* with minimal diagnosis.
- Reviewer demands more edge-case tests without a concrete failure mode material to the task → defer.
- A required criterion cannot be verified with existing tests/tools or a justified acceptance-establishing alternative → mark `BLOCKED` and ask the user to accept it unverified or authorize the tooling. Until explicitly changed, unverified required acceptance still blocks review/completion. Never build a harness on your own judgment.
- Any subagent edits outside its assigned task → stop and inspect the saved-original-baseline-to-current diff as in step 2. Revert only hunks proven to be the subagent's out-of-scope edits; never revert user work. The snapshot cannot distinguish concurrent user edits, so stop for clarification if attribution is ambiguous.

## Related files

- [implementer-prompt.md](implementer-prompt.md)
- [task-reviewer-prompt.md](task-reviewer-prompt.md)
- [references/triage-and-fix.md](references/triage-and-fix.md)
- [references/progress-ledger.md](references/progress-ledger.md)
- [references/final-verification.md](references/final-verification.md)
