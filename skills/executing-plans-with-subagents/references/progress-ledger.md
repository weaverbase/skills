# Persistent Execution Progress

The coordinator maintains one durable Markdown ledger per approved plan in the **target repository**, not the installed skill directory. It supports uncommitted work and recovery without repeating valid completed actions or resetting the [bounded review/fix policy](triage-and-fix.md). This is an execution protocol, not a general task tracker.

## Locate or initialize

1. Resolve the target repository root and approved plan file. Canonicalize the repository-relative plan path: `/` separators, no redundant `.`/`..`, consistent symlink resolution; reject paths outside the repository. Read the full plan and preserve its actual, unambiguous task IDs. Ask for clarification rather than guessing or renumbering ambiguous IDs.
2. Derive `<plan-id>` as `<basename-slug>-<path-digest-prefix>`. For the slug, remove the basename's final extension, lowercase ASCII letters, replace runs outside `a-z0-9` with `-`, trim `-`, and use `plan` if empty. Use the first 12 hexadecimal characters of a deterministic digest of the **full canonical relative path**. For example, use `printf '%s' "$PLAN_REL" | git hash-object --stdin` from the target root, checking both operations succeed. Record Git's object hash algorithm (`git rev-parse --show-object-format`) and the no-trailing-newline path encoding. Do not derive identity from plan contents, branch, working directory, or time.
3. Separately fingerprint the complete plan contents for `Plan Revision` (for example, `git hash-object --no-filters -- "$PLAN_REL"`). Hashing need not write objects. The same path with changed contents selects the same ledger.
4. Locate `.agent-work/executing-plans-with-subagents/<plan-id>/progress.md`. **Check existence and stored identity before writing.** Reuse a valid ledger for that path. If a valid existing ledger identifies a different path, extend the digest suffix one character at a time until distinct; stop if even the full digest collides. Never treat a malformed/truncated ledger or an occupied directory with unknown identity as a collision to bypass: retain it and stop for clarification. Initialize only when absent and identity is unambiguous; never truncate an existing ledger.
5. Read plan, ledger, Git status/history, and scoped implementation evidence. On an existing execution, reconcile below **before selecting work or accepting a changed revision**. On a fresh execution, initialize all tasks `PENDING`, review `PENDING`, budget `0/1`, and `Active Task: None`; persist before dispatch.

## Schema and compact example

Required headings and enums are stable:

- Task status: `PENDING`, `IN_PROGRESS`, `DONE`, `BLOCKED`.
- Active phase: `IMPLEMENTING`, `REVIEWING`, `FIXING`, `VERIFYING`; `-` when inactive.
- Review outcome: `PENDING`, `PASS`, `PASS_WITH_DEFERRED`, `BLOCKED`.
- Overall status: `IN_PROGRESS`, `DONE`, `BLOCKED`. A blocked task need not stop unaffected eligible tasks, but blocks overall `DONE`.

The illustrative tasks below are T1 (add parser) and T2 (wire command). Replace fingerprints, commands, evidence, and IDs with actual values; this is not a runtime ledger to create as a demonstration. Commit is optional (`-` for uncommitted work) and never proves acceptance.

```markdown
# Execution Progress
Plan: plans/import-command.md
Plan ID: import-command-<12-hex-path-digest>
Plan Revision: <full-plan-content-digest>
Status: IN_PROGRESS
Last Updated: <UTC timestamp>
Fingerprint Algorithm: Git <sha1-or-sha256>; path UTF-8, no trailing newline

## Tasks
| ID | Task | Status | Review | Commit |
| --- | --- | --- | --- | --- |
| T1 | Add parser | DONE | PASS | - |
| T2 | Wire command | IN_PROGRESS | BLOCKED | - |

## Active Task
Task: T2
Phase: VERIFYING
Review Stage: targeted (pending)
Fix Waves Used: 1 / 1
Next Action: run pending command check for B1, then targeted re-review

## Task Evidence
### T1
- State: DONE; phase -.
- Brief: <fingerprint of objective/acceptance/scope/dependencies>; add parser.
- Original baseline: <tree ID>, HEAD <commit>; scope src/parser.*, tests/parser.*; pre-existing edits: none.
- Result: <tree/content fingerprint>; files src/parser.js, tests/parser.test.js.
- Verification: `node --test tests/parser.test.js` PASS, 4 tests, on <result fingerprint>.
- Review: initial complete, PASS on <result fingerprint>; targeted not needed; next action: none.
- Blockers/scope: none; Fix Waves Used: 0 / 1 (default limit).
### T2
- State: IN_PROGRESS; phase VERIFYING.
- Brief: <fingerprint>; wire command to parser; depends on T1.
- Original baseline: <tree ID>, HEAD <commit>; scope src/cli.js, tests/cli.test.js; pre-existing edits: <scoped notes>.
- Result: <post-fix fingerprint>; files src/cli.js, tests/cli.test.js; uncommitted.
- Verification: initial `node --test tests/cli.test.js` FAIL for B1 on <pre-fix fingerprint>; post-fix check pending.
- Review: initial complete, BLOCK; B1 confirmed, N1 deferred; targeted pending; next action: pending B1 check.
- Authorized blockers/scope: B1 missing exit code, src/cli.js and focused test only; pre-fix baseline <tree ID>.
- Fix Waves Used: 1 / 1 (default limit); wave 1 reserved before dispatch, corrections reported complete; no further wave authorized.

## Decisions
- T2/B1: approve exit-code correction and focused test only; reviewer trace reproduced missing exit code.

## Blockers
- T2/B1: initial invocation returned 0 on invalid input; fix reported, awaiting check and targeted review.

## Deferred Findings
- T2/N1: unrelated legacy test fails on original baseline too; no fix authorized.

## Feature Verification
State: PENDING
Checked state / commands / results: -
Next Action: required command smoke check after T2; blockers/budget: none, no extra wave authorized.
```

Keep evidence for **every** task, not just the active task. Record the brief fingerprint and enough summary to compare objective, acceptance, scope, and dependencies on revision; original baseline and scope; changed files/result fingerprint; exact verification commands/results tied to that checked state; initial versus targeted review completion/disposition; approved blockers; reserved/used waves, authorized limit and any user authorization; and the next unfinished action. Keep recovery-critical facts, not a transcript. For no active task, write `None` and phase `-`.

`Active Task` summarizes its task evidence; update both together. Stop and reconcile mismatched status, review stage, next action, or counters instead of choosing a convenient version. Deferred findings and decisions are task-keyed, with short evidence sufficient to recognize them later.

## Safe persistence and baselines

- **Sole writer:** one coordinator owns all ledger updates. Pass its path/runtime subtree as out of scope to implementers and fixers; they report evidence only. Reviewers remain read-only. Runtime ledgers are not automatically committed and do not authorize `.gitignore` changes.
- Build a complete update in a temporary sibling. Validate identity, required headings/enums, all task records, matching active/task state, budgets, and evidence before replacement. Check the prior version has not unexpectedly changed. Prefer atomic replacement on the same filesystem; keep the old valid file until replacement succeeds. If atomic replacement is unavailable, preserve and verify a recoverable previous version **before** updating. Never delete the only valid copy first. Failed/interrupted writes require inspecting old, temporary, and backup versions; retain malformed files and stop if authority is ambiguous, rather than rebuilding from memory.
- Persist each task's **original baseline before initial dispatch**, including tree/commit references, initial dirty-state notes and scope. Use the [task-only snapshot instructions](../SKILL.md#0-preflight); exclude the entire runtime subtree, including tracked ledger files, temporary siblings, and backups, from both snapshots. Never alter the real index. On recovery reuse that baseline, not a snapshot of partial implementation. Review committed and uncommitted changes against the same original boundary.
- A **pre-fix baseline** is separate: capture/persist it before the reserved wave so `FIX_DIFF` covers only those corrections. It never replaces the original task baseline for acceptance or the initial review boundary. Preserve post-fix evidence too.
- Tree IDs are not permanent storage: unreferenced Git objects may be pruned. Check baseline availability; use surviving scoped diffs/content/reports if sufficient and disclose imperfect isolation. Insufficient evidence for required review is blocking. Do not create refs, automatic commits, patch archives, or a new snapshot subsystem to compensate.
- Never reset, checkout, clean, stash, or revert user work to reconcile state. Concurrent edits may not be attributable to a task; stop on material ambiguity.

## Checkpoint protocol

Persist each transition successfully **before** dispatching the next action. If persistence fails, stop advancing until the durable state is reconciled.

1. Before initial implementation: record original baseline, task `IN_PROGRESS`, phase `IMPLEMENTING`, initial review pending, budget `0/1`, and next action. Dispatch the exact approved brief with ledger writes out of scope.
2. After implementation: save changed files, result fingerprint, checks/results and optional commit; phase `REVIEWING`, stage **initial**, next action pending independent review. An implementer `DONE` is not task `DONE`.
3. After review: save the reviewer report's checked state and completion, coordinator triage, approved blockers, decisions and deferred notes **before any fix dispatch**. `PASS` maps to `PASS`; `PASS_WITH_NOTES` maps to `PASS_WITH_DEFERRED` only after triage confirms all retained findings are nonblocking. `BLOCK` is evidence to triage, not an automatic fix list or permission to discard confirmed blockers. Pending required acceptance stays blocking, even when a reviewer suggests acceptance.
4. Before a fix wave: persist approved blockers, exact authorized scope, pre-fix baseline, phase `FIXING`, next action and **increment Fix Waves Used before dispatch**. The default authorized limit is one. This reservation consumes the wave across crashes, sessions, and compaction.
5. If interrupted within a wave: inspect edits, dispatch/report state, and completed corrections. Resume only unfinished authorized work in that **same reserved wave**; do not increment again, reapply completed corrections, or treat the reservation as permission for another independent correction cycle. If execution cannot be established safely, stop for clarification.
6. After fixes: persist results, then phase `VERIFYING` for outstanding focused checks. Save completed check results before moving to `REVIEWING`, stage **targeted**, for the one pending targeted re-review. Use only the approved blockers, `FIX_DIFF`, and directly related regressions; never restart broad review.
7. Complete a task only when acceptance is met, required verification established, independent review completed, every confirmed blocker resolved, and scope preserved. An explicit justified verification alternative must actually establish acceptance, not merely note a missing check. Independent review still blocks any required acceptance left `UNVERIFIED`. Without independent review, disclose the limitation and do not mark this completion requirement met.
8. If blockers remain after the authorized wave/checks/targeted review: persist task `BLOCKED`, evidence, consumed budget and smallest next step. Only **explicit user authorization** can extend the limit by a newly finite amount; record authorization before reserving another wave. Recovery never supplies authorization. Unaffected eligible tasks may continue without bypassing blocked dependencies.
9. Before delivery: reconcile required tasks and perform the existing [bounded feature verification](final-verification.md). Save its checked state, exact commands/results, pending/completed actions, blockers and applicable consumed/authorized budget in `Feature Verification`. Carry forward the existing bounded budget when corrections are needed; final verification is not a fresh or unlimited allowance. Preserve the optional final-review policy. Overall `DONE` requires all required tasks and feature acceptance/checks; save overall status and report honestly.

## Resume and plan revisions

Run on **every new execution session and context recovery**, not from conversation memory alone. Compare plan revision, Git status/history, saved result/verification/review evidence, task dependencies, and any in-flight report before dispatch.

| Saved task state | Reconciliation and next action |
| --- | --- |
| `DONE` | Inspect only enough task-relevant evidence to detect invalidation; skip still-valid implementation, checks and reviews. A changed HEAD alone does not invalidate completion, and a commit alone does not prove it. Missing/changed relevant evidence requires narrow reconciliation, not automatic reimplementation. |
| `IN_PROGRESS` | Resume earliest unfinished required action: `IMPLEMENTING` preserves partial/staged edits; `REVIEWING` resumes only pending initial **or targeted** review; `FIXING` continues authorized unfinished corrections within the saved wave; `VERIFYING` runs only outstanding checks. |
| `BLOCKED` | Retain blockers and consumed budget, including on a second session. Retry only with new evidence, appropriate authorization, or an approved approach within remaining budget. |
| `PENDING` | Dispatch only when required dependencies permit; never silently pass a blocking dependency. |

If the complete plan fingerprint differs from `Plan Revision`, compare saved task briefs/dependencies to identify changed, added, removed and dependency-affected tasks. Preserve unaffected completions, reviews, deferred dispositions and budgets; revisions do not reset consumed waves. Record approved disposition before invalidating affected evidence. Do not silently drop removed unfinished tasks or guess renumbered mappings. Escalate material scope, acceptance or dependency ambiguity. Update the **accepted** `Plan Revision` only after reconciliation has completed, so a crash cannot acknowledge unprocessed changes.

Also reconcile `Feature Verification`: reuse valid completed results; resume only outstanding actions or narrowly recheck invalidated evidence. A deferred unrelated failure remains deferred unless new evidence or explicit authorization changes its disposition; it is not a new fix cycle.

Fresh example: `plans/import-command.md` resolves one ledger; initialize once, then save T1's baseline and `IMPLEMENTING` checkpoint before dispatch. Recovery example: the table above resumes T2's outstanding command check, saves its result, then requests only targeted review of B1; it neither reruns T1 nor consumes a second wave. If the review completed but its checkpoint did not, inspect the surviving report and checked state before recording completion or deciding what remains unfinished.

## Limits and fail-safe behavior

- Repeated recovery with unchanged plan/repository/evidence must not dispatch completed implementation, review, fixes, or checks. In-flight actions may require inspection: exactly-once agent dispatch/completion across a crash is **not guaranteed**.
- One coordinator per ledger; atomic replacement is not distributed locking or a concurrency guarantee. Stop on conflicting writers. Worktrees/clones do not automatically share uncommitted runtime state.
- Stable paths identify plans. Renames/relocations require explicit reconciliation; there is no automatic migration. Fingerprints detect changes, not semantic equivalence or acceptance.
- Markdown is maintained with existing agent/filesystem/Git capabilities, not a transactional engine or schema validator. Missing evidence, ambiguous identity, malformed state and unavailable safe persistence fail closed. No external skill dependencies, runtime tooling, background services, automatic commits, or changes to the user's staging are introduced.
