# Coordinator Triage and Bounded Fix Policy

**Reviewer output is evidence, not an automatic to-do list.** The coordinator classifies every finding before any fixer is dispatched.

## Classification

| Class | Criteria | Action |
| --- | --- | --- |
| BLOCKER | Confirmed unmet acceptance criterion; introduced regression; security or data-integrity flaw (including a race) on an in-scope path, reproduced with existing tools or shown by direct code trace; introduced failure preventing required verification | Approve smallest safe fix wave |
| UNKNOWN | Material issue whose cause/impact is not established | Run smallest diagnostic; if required behavior remains unverified, keep task BLOCKED |
| DEFER | Suspected race or edge case without reproduction or trace; optional edge case without material impact; pre-existing unrelated failure; refactor; new abstraction; test framework redesign; general maintainability advice | Record brief note, do not dispatch fix |

Important: A bug found outside the diff can still be critical if the feature would expose it (e.g. an auth bypass). Never suppress a demonstrated safety defect merely because it was not named in the plan.

## Failure provenance before fixing integration tests

1. **Introduced:** reproducible due to the current change → blocking if it affects required behavior.
2. **Pre-existing:** demonstrated on baseline/unaffected paths → do not fix inside this task, unless feature cannot be verified and no safe alternative exists; then report blocked verification.
3. **Environment:** dependency unavailable, credentials missing, flaky service → retry/diagnose narrowly, record limitation.
4. **Unknown:** no adequate evidence → do not claim passing verification; resolve the smallest critical uncertainty or escalate.

## Default maximum: one fix wave

Collect approved blockers into **one cohesive instruction**:

```text
Fix ONLY these coordinator-approved BLOCKERs for task {ID}:
{NUMBERED_FINDINGS_WITH_EVIDENCE}

Working directory and baseline: {WORKDIR_AND_BASELINE_NOTES}
Out of scope / do not touch: {OUT_OF_SCOPE}

Constraints:
- No new feature requirements, broad refactors, new test harness or infrastructure, or unrelated fixes.
- If the fix cannot be verified with existing tests/tools, stop and report BLOCKED; do not build tooling.
- Preserve unrelated user changes.
- Use the smallest safe correction and focused tests.
- Report what changed, exact test results, and unresolved blockers.
- Do not spawn subagents or request your own review.
```

Then dispatch **one targeted re-review**:

```text
TASK BRIEF / ACCEPTANCE CRITERIA:
{TASK_BRIEF}

APPROVED BLOCKERS:
{NUMBERED_FINDINGS_WITH_EVIDENCE}

FIX DIFF AND TEST RESULTS:
{FIX_DIFF}
{FIX_TEST_RESULTS}

Check ONLY whether these blockers were resolved and whether these specific
fixes introduced directly related regressions. Read-only. Do not start a
new broad review. Report PASS or BLOCK with concrete evidence.
```

If blockers remain after the wave: `BLOCKED`, provide evidence and smallest viable next step. A further wave requires an explicit coordinator/user decision and a newly stated finite budget; never repeat automatically.

## Ledger (in conversation or a project file when useful)

| Task | Status | Required checks | Blockers | Deferred observations |
| --- | --- | --- | --- | --- |
| T1 | REVIEWING | command + result | finding ID | note ID |

Do not represent deferred notes as mandatory tasks. Do not let deferred notes reenter the work queue without authorization.
