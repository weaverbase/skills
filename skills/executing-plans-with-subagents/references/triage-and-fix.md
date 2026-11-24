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

## Default maximum: three fix waves

Collect approved blockers into **one cohesive instruction per wave**. Within the saved limit, the coordinator may dispatch without asking the user again. Between waves, re-triage only unresolved confirmed in-scope blockers or blocking regressions directly caused by the previous corrections. Each new wave needs an evidence-backed next correction; a remaining slot is not a reason to repeat a failed approach under unchanged conditions, broaden the review, or change approved behavior. A missing verification prerequisite alone does not justify another wave.

Before dispatch, the coordinator checkpoints approved blockers, exact scope, the wave's pre-fix baseline, phase `FIXING`, and the incremented `Fix Waves Used` in the [persistent ledger](progress-ledger.md). Reserve within the saved authorized limit (default three for new tasks); failed persistence stops dispatch. Pass the ledger/runtime subtree as out of scope to fixers. On recovery, inspect edits and reports and continue only unfinished corrections in the same reserved wave; do not reset the counter, repeat completed fixes, or start another wave under the same reservation. Ambiguous execution state requires clarification.

On **wave 3**, use `escalation-fixer` if available and its use criteria are met (the change has failed review more than twice: typically the initial review and two targeted re-reviews). Pass the full feedback from every review round, redacted as needed, and what each previous fix tried, along with focused check results and the exact remaining authorized blockers and scope. Earlier findings are context, not a new fix list. If the specialized fixer is unavailable or ineligible, use the normal fixer under the same constraints; do not add review rounds just to qualify. The escalation fixer does not replace independent re-review.

```text
Fix ONLY these coordinator-approved BLOCKERs for task {ID}:
{NUMBERED_FINDINGS_WITH_EVIDENCE}

Working directory and baseline: {WORKDIR_AND_BASELINE_NOTES}
Out of scope / do not touch: {OUT_OF_SCOPE}

Constraints:
- No new feature requirements, broad refactors, new test harness or infrastructure, or unrelated fixes.
- Preserve approved behavior and required data. If the correction needs a material scope change, stop and report it.
- If the fix cannot be verified with existing tests/tools, stop and report BLOCKED; do not build tooling.
- Preserve unrelated user changes.
- Use the smallest safe correction and focused tests.
- Report what changed, exact test results, and unresolved blockers.
- Do not spawn subagents or request your own review.
```

Persist fix results and phase `VERIFYING`; run outstanding focused checks and save their results, then persist phase `REVIEWING` / targeted before dispatching **one independent targeted re-review for this wave**. Save its completion before advancing. Never rerun a completed initial review on recovery.

`FIX_DIFF` compares a saved **pre-fix** snapshot with the post-fix snapshot, not the original task baseline. Use the runtime-excluding `snap` from [preflight](../SKILL.md#0-preflight), from the resolved target repository root:

```sh
PRE_FIX=$(snap) || exit 1      # persist before reserving/dispatching this wave
# After the authorized corrections (reuse saved PRE_FIX on recovery):
POST_FIX=$(snap) || exit 1
git diff "$PRE_FIX" "$POST_FIX" -- || exit 1
```

The original task baseline remains unchanged for task acceptance and the initial review boundary; pre-fix evidence only isolates the authorized corrections.

Targeted review instruction:

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

If blockers remain after the wave and targeted re-review, continue only if triage identifies an evidence-backed in-scope correction and the saved budget allows it. Otherwise persist `BLOCKED`, evidence, consumed budget and smallest viable next step. Extending an exhausted budget requires **explicit user authorization** and a newly stated finite limit recorded before dispatch. Recovery grants no additional authorization.

## Persistent decisions and evidence

Use the mandatory [progress-ledger protocol](progress-ledger.md), not a conversation-only table. The coordinator alone persists each task's review disposition, approved blockers/scope, verification evidence, next action, deferred notes, and used/authorized wave budget. `PASS_WITH_NOTES` becomes `PASS_WITH_DEFERRED` only after coordinator triage confirms the retained findings are nonblocking; `BLOCK` remains evidence for triage, not an automatic fix list. Required acceptance left `UNVERIFIED` still blocks independent review and task completion, even if a missing check was documented; a justified alternative must establish acceptance.

Do not represent deferred notes as mandatory tasks. Preserve their disposition across recovery; new evidence or explicit authorization is required to reconsider them, not merely another session or a pre-existing unrelated failure. See the same ledger protocol for feature-verification checkpoints; final delivery creates no new fix budget.
