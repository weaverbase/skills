# Final Feature Verification (No Unbounded Final Review)

After completing all unblocked tasks:

1. Compare implemented behavior with **feature-level acceptance criteria**, not imaginary future requirements.
2. Run relevant integration or end-to-end checks available in the project **once** (and a focused rerun after necessary corrections). Respect existing CI requirements.
3. Classify failures as introduced, pre-existing, environment, or unknown. Do not turn infrastructure/test framework problems into an unapproved subproject.
4. Review cross-task wiring, API contracts, migrations, security/data-integrity boundaries actually touched by this feature. Avoid repo-wide best-practices audits.
5. For confirmed blockers, use the same coordinator triage policy and a finite, explicitly scoped fix wave; escalate if unresolved.
6. Report honest status. A partially verified feature is not `DONE` when mandatory criteria are unverified.

## Delivery format

```text
FEATURE STATUS: DONE | PARTIAL | BLOCKED
IMPLEMENTED: <scope summary>
ACCEPTANCE: <criterion-by-criterion outcome>
VERIFICATION: <commands, PASS/FAIL/NOT RUN, important limitations>
BLOCKERS: <none or specific issue with evidence>
DEFER: <brief list of non-blocking follow-ups, if any>
SCOPE CHANGES: <none or explicit approvals>
```

A final reviewer can be dispatched when meaningful (for cross-task risks), but only with this scope and **one bounded pass**. Do not automatically use any other code-review skill.
