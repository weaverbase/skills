# Weaverbase Skills

Public agent skills for applying practical engineering defaults across projects. The guidance is general-purpose and not limited to any specific codebase.

## Included Skills

Each skill is self-contained and can be installed independently. None requires another skill or a third-party skill collection.

### `writing-commit-messages`

Use when writing, amending, or reviewing git commit messages.

- Format messages as `<type>(<optional scope>): <short description>`
- Add an optional body and footers after a blank line

### `designing-cohesive-services`

Use when adding business logic to the service layer or deciding how it is grouped.

- Choosing between a cohesive service method, standalone function, query module, and pure helper
- Grouping operations by shared state, resource, or invariant
- Avoiding pass-through wrappers, class-per-action services, and mandatory layers

### `keeping-entry-points-thin`

Use when implementing or changing HTTP handlers, server actions, CLI commands, jobs, or workers.

- Business logic in the service layer; entry points validate, call the service, and map output
- Litmus: callable unchanged from CLI, API, and job means it belongs in the service layer
- Services take domain types, return typed results, and raise domain errors, never transport exceptions
- Interfaces map domain errors to HTTP status or exit code; a new interface never changes a service

### `handling-transactions-and-side-effects`

Use when changing transactions, external writes, event delivery, retries, idempotency, or recovery from partial failure.

- Let operations own short transactions and race-sensitive checks
- Publish after commit and use durable intent or an outbox when needed
- Make automatic retries safe, leave user-initiated retry policy to the project, and verify external outcomes before finalizing

### `validating-at-boundaries`

Use when defining request/response schemas, parsing external payloads, forms, or environment configuration.

- Put shape and format constraints on schemas at real boundaries
- Pass typed values inward while keeping current-state business rules in operations
- Use Pydantic v2 or Zod and derive related shapes without duplicating fields

### `writing-python`

Use when changing Python code, FastAPI dependencies, Python contracts, PDF processing, logging, or tests.

- Use modern type hints, dictionary merging, and `pytest`
- Prefer `pypdfium2` for reading and rendering PDFs; avoid `pdf2image` and PyMuPDF for license reasons
- Prefer discoverable nominal contracts over internal `Protocol` abstractions
- Use async FastAPI database access and apply the bundled logging template as app-local configuration

### `writing-typescript`

Use when changing TypeScript/JavaScript, React/Next.js, Node.js file operations, or tests.

- Use ESM, `const`/`let`, `async`/`await`, and `vitest` or `jest`
- Follow function-component and App Router conventions for new work
- Use `node:` built-in imports and promise-based filesystem APIs

### `writing-rust`

Use when changing Rust error handling, string ownership, or async runtime code.

- Propagate errors with `Result` and `?`
- Borrow strings where practical instead of allocating or cloning unnecessarily
- Follow current Tokio patterns without production `unwrap`/`expect`

### `writing-container-configs`

Use when changing Dockerfiles or Docker Compose configuration.

- Use specific image tags, multi-stage builds, and non-root runtime users
- Follow the current Compose Specification without a top-level `version`
- Use mappings for environment variables and labels, quoting YAML-sensitive values

### `designing-sql-schemas`

Use when changing SQL schemas, migrations, JSON/timestamp columns, or database query APIs.

- Prefer PostgreSQL `JSONB` and timezone-aware timestamps
- Add migrations rather than editing applied ones, and prefer additive steps over data-losing ones
- Default to async database APIs (context-aware calls in Go) and parameterize SQL

## Works Well Together

For an endpoint backed by a database, service design, thin entry points, boundary validation, and transaction handling cover different decisions. Add the relevant language skill for stack conventions. These combinations are suggestions, not dependencies.

## Migration

`using-weaverbase` has been replaced by the focused skills above and is no longer provided. Existing users should install the full set or select the skills relevant to their work. The former shared references now live inside their owning skills.

`commit-messages` has been renamed to `writing-commit-messages`. Existing users should reinstall it under the new name, for example `npx skills add weaverbase/skills@writing-commit-messages`.

## Install

Install the full skill set:

```bash
npx skills add weaverbase/skills
```

Install only `designing-cohesive-services`:

```bash
npx skills add weaverbase/skills@designing-cohesive-services
```

Install for a specific agent:

```bash
npx skills add weaverbase/skills --agent codex
```
