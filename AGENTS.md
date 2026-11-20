# Repository Guide

## Purpose and layout
- This is a Markdown-first collection of reusable agent skills, not an application.
- Each `skills/<skill-name>/SKILL.md` is independently installable and self-contained.
- Supporting guidance and templates belong alongside the owning skill, usually in `references/`.
- `README.md` is the public skill catalog and installation guide.

## Editing conventions
- Read the target skill and applicable references before editing; follow their existing structure.
- Keep `SKILL.md` YAML frontmatter valid: `name` matches the directory; `description` states when to use the skill.
- Write concise, actionable guidance with concrete examples. Keep detailed stack-specific material in linked references.
- Do not introduce implicit dependencies on other skills or third-party skill collections.
- Apply rules to new or requested changes; do not migrate unrelated working content or expand scope.
- Keep relative links valid. Update README entries and affected links when adding, renaming, or removing skills.
- Preserve existing user changes. Do not add runtime artifacts, dependencies, or testing infrastructure without a task requirement.

## Verification and delivery
- Run `git diff --check`; inspect the full intended diff and new files explicitly, since ordinary `git diff` omits untracked files.
- Check changed frontmatter, local links/anchors, example correctness, and consistency between a skill and its references.
- There is no shared build or test suite. Execute relevant examples when practical; distinguish executed checks from prose walkthroughs and checks not run.
- Summarize changes, verification results, and concrete limitations. Do not stage or commit unless explicitly requested.
