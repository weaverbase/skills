---
name: writing-implementation-plans
description: Use when writing, repairing, or revising a multi-step implementation plan that needs a reviewable design, bounded tasks, and verifiable acceptance. Not for executing plans, tracking progress, or making straightforward edits.
---

# Writing Implementation Plans

Write a Markdown plan another engineer can review or implement one task at a time. The plan records intended scope, decisions, and planned checks, not execution state. Use the project's format or the [optional template and example](references/plan-template.md). The content matters, not the headings.

## Ground the design

- Read the request or spec, repository instructions, relevant code and tests, current state, and existing verification entry points. For repairs or revisions, read the current plan and compare it with the request and repository evidence.
- Summarize the findings that justify the design in the plan itself. Separate observed facts from material assumptions, risks, and open questions. Settle details the repository evidence answers without asking. Raise material choices about scope, data, interfaces, security, or acceptance with the user as short options with a recommendation. Use safe, reversible defaults only for minor choices, and state them. Keep the plan a draft while a material choice is unresolved.
- Map the known paths or components and their responsibilities before splitting tasks. Mark new paths as proposed. Follow the existing architecture rather than restructuring to fit a template. Record shared constraints once and carry the applicable ones into any task brief shared on its own.

## Preview before finalizing

Once the approach is grounded and still open to change, send the user a short, visible preview with the intended outcome, high-level approach, proposed task breakdown, what is in and out of scope, and any material assumption or trade-off. For a revision, call out what materially changed. A section in the plan or a summary at final handoff does not count as this preview.

Treat the preview as a point for discussion. Let the user respond before finalizing unless they asked you to proceed, incorporate their feedback, and preview again if the design or scope changes materially. Do not ask for sign-off on routine details, and do not treat the preview or the user's feedback as permission to implement. If self-review changes anything material, surface the change to the user before finalizing.

## Define tasks and evidence

- Order bounded tasks around independently reviewable results: a reviewer should be able to reject one deliverable without rejecting its neighbor. Fold setup, tests, and supporting docs into the task that needs them. Split unrelated features into separate plans if they can be accepted and delivered independently.
- Give each task a unique, stable ID, objective, dependencies by ID (or none), affected paths or components, and explicit scope boundaries. Pin material interfaces: name the provider and consumer and agree on relevant signatures, types, values, and error behavior. Use short steps where sequence matters; leave routine implementation choices to the implementer.
- State observable acceptance criteria at task and feature level, including material failure behavior at the layer that owns it. For each task, name focused checks and what should be observed. Add a feature-level integration check when tasks interact, with its expected result. Use verified repository commands or concrete manual procedures with any needed inputs and preconditions.
- If a required check has no feasible method, name the unverified criterion, the missing prerequisite or decision, and why it blocks readiness. Do not invent a harness or quietly weaken acceptance. Describe checks as planned, not passed.

## Self-review and hand off

Trace each material requirement to an owning task and a check. Read each brief on its own: look for vague steps, missing constraints, missing or out-of-order dependencies, and mismatched producer/consumer interfaces. Check that credible failures implied by the scope have an owning check. Remove details that do not capture a decision. Confirm the final design and scope match what was agreed with the user, and point out anything that changed after that discussion.

Follow the user's or project's plan-file convention. For handoff to an executor that binds plans to a repository and worktree, save at a stable Markdown path inside the intended repository. If there is no convention, `plans/<feature-slug>.md` is one option. Give the saved repository-relative path and intended workspace (repository or specific worktree) at handoff. Record any execution choices the user stated, such as a required environment or execution approach, in the handoff notes; do not assume a runtime or require another skill.

On revision, keep the saved path and IDs of surviving logical tasks. Assign fresh IDs to new tasks; explain changed and removed tasks, any replacements or mappings, and affected dependencies. Do not recycle IDs or change the plan just to report progress.

At handoff, disclose any readiness blockers. Do not claim approval: it comes from the user or a user-authorized workflow, not the plan or its preview. Do not start implementation or create progress files, baselines, or an execution ledger.
