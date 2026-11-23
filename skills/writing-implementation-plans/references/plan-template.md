# Optional Plan Template

Use any layout that carries this information. Omit fields that do not apply; say so explicitly for task dependencies. The user-facing design and scope preview is a separate message before finalization, not a field to fill in.

```markdown
# <Feature>

## Goal and scope
<Intended outcome, in/out of scope, material assumptions or risks>

## Design and file responsibilities
<Repository findings and paths that justify the design; mark proposed paths>
<Shared constraints to carry into applicable task briefs>

## Feasibility and unknowns
<Load-bearing assumptions: each with its evidence (cited doc, version, code path), or its disposition: gated by task <ID>, or accepted with fallback>

## Feature acceptance
<Observable outcomes and planned integration checks with expected results>

## Tasks
<Put de-risking tasks first; dependents list them in "Depends on".>

### <Stable ID> - <Reviewable result>
- Objective and boundaries: <deliverable and what it must not change>
- Depends on: <task IDs or none>
- Files or components: <paths and responsibilities>
- Interfaces: <exact material contract provided or consumed, with its owner>
- Acceptance: <observable success and material failure behavior>
- Focused verification: <planned check, prerequisites, expected observations>

### <Stable ID> - Decide <question> (de-risking task, if needed)
- Question and why it is load-bearing: <what must be true, and which tasks or interfaces break if it is not>
- Method and timebox: <what to read, run, or prototype; inputs and environment; what to leave unchanged>
- Evidence to produce: <observations, measurements, or a user decision to report>
- Decision rule: <outcome A -> proceed with tasks <IDs>; outcome B -> use fallback <design>; otherwise stop and revise the plan>

## Handoff notes (if applicable)
<Execution choices the user stated (for example, required environment or execution approach); unresolved required checks and why they block readiness>
<For revisions: material changes, added/removed/mapped IDs and affected dependencies>
```

## Example: Limit CLI report rows

Illustrative repository and planned checks only. The paths, fixture, and test runner below describe the example project, not this skill repository.

**Goal and scope:** Add an optional `--limit <value>` to the existing report CLI. With no flag, preserve the full report. Do not change CSV parsing, output format, or other options.

**Design and files:** Existing `bin/report.js` owns CLI argument handling and output. Existing `test/report-cli.test.js` uses Node's built-in runner and `test/fixtures/report.csv`, which yields `alpha`, `beta`, and `gamma` on separate lines in that order. Put conversion in proposed `lib/report-options.js`; keep stderr and exit handling in the CLI. Keep the existing runner and add no dependencies.

### T1 - Parse a report limit

- **Depends on:** None.
- **Files and boundaries:** Define the parser in proposed `lib/report-options.js` and add focused tests in proposed `test/report-options.test.js`. Do not change CLI behavior in this task.
- **Provides:** `parseLimit(raw: string): number | undefined` from `lib/report-options.js`. Accept ASCII decimal digits only, with no leading zeros and a value from `1` to `100`; return `undefined` otherwise. No I/O.
- **Acceptance and focused check:** `node --test test/report-options.test.js` should pass: `"1"`, `"2"`, and `"100"` return numbers `1`, `2`, and `100`; `"0"`, `"101"`, `"02"`, and `"2x"` return `undefined`.

### T2 - Apply the limit at the CLI

- **Depends on:** T1.
- **Files and boundaries:** Update flag handling and report output in `bin/report.js`, with CLI coverage in `test/report-cli.test.js`. Do not change the parser contract, CSV format, or other options.
- **Consumes:** `parseLimit(raw: string): number | undefined` from `lib/report-options.js` (T1). Pass a supplied flag value to the parser. A missing value or an `undefined` result is a usage error.
- **Acceptance and focused check:** `node --test test/report-cli.test.js` should pass: on the three-row fixture, `--limit 2` prints the first two rows in order and exits `0`; an omitted flag still prints all three and exits `0`; `--limit 101` and a flag with no value both exit `2` with empty stdout and `Invalid --limit` on stderr.

**Feasibility note (illustrative):** Limit bounds `1`-`100` and the exit code `2` for usage errors were confirmed in the existing CLI tests and `bin/report.js`. If the CLI's argument library had not been confirmed to pass `--limit` with no value through to the handler, add an early task: "Decide how a valueless flag reaches `bin/report.js`", with a method (a minimal call against the installed version), the evidence (observed argument value), and a decision rule (handle it in the parser, or change the T2 contract) before T2.

**Feature acceptance and integration check:** From the example repository root with Node available, run `node bin/report.js test/fixtures/report.csv --limit 2`. Expect exit `0`, empty stderr, and exactly `alpha` then `beta` on stdout, one row per line. Repeat with `--limit 101`: expect exit `2`, empty stdout, and the line `Invalid --limit` on stderr. Repeat without the flag: expect exit `0` and all three rows in their original order.
