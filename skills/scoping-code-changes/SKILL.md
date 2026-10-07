---
name: scoping-code-changes
description: Use when starting, reviewing, or finishing any code or configuration change, especially when the request is ambiguous, "quick", or urgent; when a change might alter a public API, CLI, config, event, or database contract; when it might delete data, add a dependency, or touch auth, secrets, or crypto; when deciding what tests to add or run; or when writing the final summary of what was done and verified.
---

# Scoping Code Changes

## Overview

Make the smallest coherent change that does what was asked, prove it works, and say honestly what was and was not checked.

**Core principle:** Scope, safety, and evidence are not negotiable under time pressure. Urgency, "it is only a small change", and "we can clean it up later" change how quickly you work, not what you are allowed to skip. A request for speed is not a waiver of the safety baseline.

## Precedence

When instructions conflict, follow them in this order:

1. Constraints from the system, platform, or developer environment you are running in (permissions, tool limits, policies).
2. Explicit task instructions from the user, within the authority they have over the work.
3. Repository guides (`AGENTS.md`, README, contributing docs).
4. Conventions of the code you are changing.
5. Defaults from skills or general habit.

A higher level wins only within its own authority. A task instruction cannot grant permissions the environment withholds, and "do it quickly" does not authorize leaking a secret or disabling authentication. If a higher level conflicts with a lower one, follow the higher one and mention the conflict in your report.

Read the relevant code and its tests before editing. Do not guess at conventions you could have read.

## Scope

- Make the smallest coherent change that satisfies the request, including the tests and docs that change must carry.
- Do not refactor, rename, reformat, or reorganize unrelated working code. Mention it in the report instead.
- Apply new rules to new or requested changes only. Never require migrating existing working code to a new convention.
- A change the user explicitly asked for is in scope, even if it is large or touches a public contract. Do the work, and state in the report that it was requested and what it affects.

## Stop and Report Options

Stop before making the change, describe what you found, and offer concrete options when any of these hold:

- The requirements admit materially different behaviors and the choice is not clear from the request, the code, or the repo guides.
- A public API, CLI, config format, event, message, or database contract would change and the request did not ask for it, or it is unclear whether the request covers it.
- Data may be destroyed or become unrecoverable (dropping columns, deleting records, rewriting files in place) and that was not clearly requested.
- A new dependency or a new service/infrastructure component is needed.
- Existing tests contradict the request.
- Authentication, authorization, cryptography, or secrets handling would change beyond what was asked.

Stopping is for **unrequested or unclear** changes. Do not stop for every change that touches an API: if the user explicitly asked for the contract change, proceed, keep the change to what they described, and in the report name the contract that changed and the callers that may be affected.

When you stop, give the smallest useful question: what you found, the two or three options, the consequence of each, and which you would pick.

## Safety Baseline

These hold for every change, regardless of urgency:

- Never hardcode or log secrets, tokens, passwords, or personal data.
- Never build shell commands from untrusted strings; pass arguments as a list or use an API instead.
- Validate file paths and uploads (location, size, type) before using them.
- Do not weaken authentication, authorization, TLS verification, CORS, or rate limits to make a change or a test work. If one of them is in the way, report it.

## Verification

- Test observable behavior and the important failure cases, not internal call sequences.
- A behavior change or bug fix needs a focused test that fails without the change and passes with it. Manual checking and urgency are not substitutes.
- Do not weaken, delete, or skip a valid test to get a green result. If a test is wrong, say why and fix it deliberately.
- Do not write pass-through tests that exist only to raise coverage.
- Run the narrow tests relevant to your change first, then the repository's documented checks (tests, lint, type check, build) that cover what you touched.
- Never claim a check passed unless you ran it and saw the result.

## Report

End with a short report that contains:

- What changed and why.
- Checks run, with their results.
- Checks not run, and why (for example: needs credentials, too slow, not available).
- Assumptions made and open questions.
- Confirmation that no unrelated files were modified and that no security or transaction guarantees changed, or a clear statement of what did change.

## Example

Request: "Add an optional `note` field to the order creation endpoint."

Good:

- Reads the endpoint, its schema, and the existing order tests; follows the naming used there.
- Adds the optional field, the validation, the persistence of the field, and tests for "with note", "without note", and "note too long".
- Does not rename neighboring fields or reformat the module.
- Notices that persisting the note needs a new column. The request implied this, so it adds a new migration instead of editing an applied one.
- Runs the order tests and the repository's lint and type check. The full integration suite needs a database that is not available, so it is reported as not run.
- Reports: the field is optional so existing clients are unaffected; checks run with results; integration suite not run and why; no unrelated files changed.

Bad:

- Also renames `qty` to `quantity` "while in there", changing the response of an endpoint that other clients use.
- Hardcodes a test API key in the fixture to get the test passing.
- Skips a failing existing test because "it looks unrelated".
- Ends with "Done, all tests pass" without having run them.

## Common Mistakes

| Mistake | Correct move |
| --- | --- |
| Treating "just do it quickly" or "skip the formalities" as permission to hardcode secrets, drop validation, or loosen auth | Do the compliant version, which is usually only slightly slower. If an explicit, specific instruction conflicts with a repo rule, follow it within its authority and note it in the report. |
| Stopping to ask about every contract change, even one the user explicitly requested | Proceed with what was asked, keep to its scope, and describe the affected contract in the report. |
| Silently widening a change to include an unrequested API, config, or schema change | Stop and report options. |
| Refactoring or renaming nearby code because it looks untidy | Leave it; list it in the report as a follow-up. |
| Editing an already applied migration or overwriting data in place | Add a new change, and stop first if data could be lost. |
| Making a failing test pass by loosening it, mocking away the thing under test, or skipping it | Fix the code, or explain why the test is wrong and change it deliberately. |
| Adding a test that only calls a function and asserts it was called | Assert an observable result or failure instead. |
| Saying "tests pass" from memory, from a previous run, or from reading the code | Run the check and report its actual result. |
| Reporting only successes | List checks not run and assumptions made. |
| Disabling TLS verification, CORS, or rate limits to get a local run working | Fix the configuration of that environment, or report the blocker. |

## Red Flags

Stop and re-read this skill if you think or write:

- "This is urgent, so I will skip the tests and verify manually."
- "The user asked for it fast, so security checks can wait."
- "While I am here, I will clean up this other code."
- "It is a small API change, nobody will notice."
- "I will assume what they meant and mention it later."
- "The test is flaky or wrong, so I will skip it" (without evidence).
- "It should pass, so I will say it passes."
