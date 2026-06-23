# Weaverbase Skills

Public agent skills for applying practical engineering defaults across projects. The current skill is named `using-weaverbase`, but its guidance is general-purpose and not limited to any specific codebase.

## Included Skills

### `using-weaverbase`

Use when creating or modifying code, reviewing changes, or choosing architecture, validation, naming, or Docker Compose patterns.

It covers defaults for:

- Keeping business logic callable from API, CLI, UI, workers, and background jobs
- Validating data at boundaries with Pydantic or Zod
- Passing strict typed values inward instead of raw `dict`, `unknown`, or `any`
- Returning structured result types from services
- Naming by intent instead of generic type suffixes
- Mapping errors to safe user-facing messages at boundaries
- Adding or updating focused tests when behavior changes
- Using current Docker Compose syntax

## Install

Install the full skill set:

```bash
npx skills add weaverbase/skills
```

Install only `using-weaverbase`:

```bash
npx skills add weaverbase/skills@using-weaverbase
```

Install for a specific agent:

```bash
npx skills add weaverbase/skills --agent codex
```
