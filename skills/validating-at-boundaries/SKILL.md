---
name: validating-at-boundaries
description: Use when defining or changing request bodies, query params, form input, environment or config parsing, external API responses, message payloads, or JSON data, or when adding Pydantic models, BaseSettings, Zod schemas, z.infer types, safeParse calls, unknown/any/raw dict inputs, or ad-hoc shape checks in services, handlers, or React components.
---

# Validating at Boundaries

## Overview

A schema is the single source of truth for both the shape and the validation of a concept. External data is checked once, at the boundary where it enters (or leaves) the system. Everything past that point works with validated, typed values and does not re-check them.

**Core principle:** Put structural validation on a schema at the boundary. Keep state-dependent rules in the service layer. Pass typed values inward.

Apply this to new or requested changes. Do not migrate working code unprompted.

## Stack References

When writing or reviewing schema code, read the reference for the stack:

- Python and Pydantic v2 (`BaseModel`, `ConfigDict`, `BaseSettings`, `SettingsConfigDict`): [references/pydantic.md](references/pydantic.md)
- TypeScript and Zod 4 (schemas, `z.infer`, React boundaries, env parsing): [references/zod.md](references/zod.md)

## Who Validates What

| Owner | Validates |
| --- | --- |
| Schema at the boundary | Structure: types, required fields, length, format, unknown keys, coercion |
| Service layer | Domain rules that need business context, persistence, authorization, or current system state |
| Components and presentational code | Nothing about external shape. They receive typed props |

## Rules

1. **Define each concept's shape once.** The schema is the source of the type; do not write a second hand-made type or checker next to it. When the project has not chosen a library, use Pydantic v2 in Python and Zod 4 in TypeScript; otherwise follow the project's choice.
2. **Validate at boundaries:** request bodies, query params, environment variables, external API responses, form inputs, message payloads, and JSON columns or tables written by other systems.
3. **Services accept validated, typed models**, not raw `dict`, JSON, `unknown`, or `any`.
4. **Do not re-check what the schema guarantees.** If `name` has a maximum length of 100 on the schema, the service does not check the length again. Branching on legitimately optional or empty values (a nullable nickname, an empty list) is domain logic and stays in the service.
5. **Derive create and update variants from shared definitions.** Subclass a Pydantic base or reuse constrained field aliases; derive Zod variants with `.extend()`, `.partial()`, `.pick()`, or `.omit()`. Do not copy constraints into separate definitions.
6. **Unknown keys follow the project.** When the project has no convention, reject unknown keys on inbound request bodies so typos surface. Ignore them on external API responses so new upstream fields do not break you.
7. **Validate outbound data.** Return explicit response models so internal fields, such as password hashes or internal IDs, cannot leak.
8. **Surface validation errors at the boundary** as 400 or 422 responses or as form errors. Services raise domain errors, not validation errors.
9. **Trust your own database.** Map rows to typed models in the persistence code that reads them; do not re-parse them in services.
10. **Validate configuration.** Read environment variables through a typed schema (Pydantic settings, a Zod schema for `process.env`), parse once at startup, and pass typed values inward.

## Frontend Data

Parse external data where it enters the UI: route loaders, server actions, API-client adapters, form resolvers, or explicitly named boundary or container modules. Reusable and presentational components receive typed props.

Components must not accept `unknown` or `any` for external data, call `parse` or `safeParse`, run `if (!value?.id)`-style shape guards, or hide invalid external data by returning `null`. If external data fails validation, surface the failure at the boundary: an error state, a form error, or a thrown error that an error boundary handles.

## Example

Good flow: the boundary validates once; the service receives a typed value and enforces only domain rules.

```text
request body --> boundary: parse with CreateInvite schema (400/422 on failure)
             --> service: invites.create(invite: CreateInvite)
                    - domain rule: reject a duplicate address (ConflictError)
             --> boundary: map ConflictError to 409; serialize InviteOut
```

Bad flow: the service takes `dict`/`unknown`, checks `"@" in email` and the name length itself, and returns the internal record. The shape then lives in three places and nothing stops internal fields from leaking.

Full code for both stacks is in the references.

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "Validate directly in the service or component; the check is small." | Put it on the schema at the boundary. |
| "Accept a raw `dict`/`any` now; add a schema later." | The shape is the contract. Define it once now. |
| "The model is validated, but re-check the length to be safe." | Do not re-check what the schema guarantees. |
| "`safeParse` in the component and return `null`." | Parse at a real boundary and surface the failure deliberately. |
| "Copy the create fields into the update model." | Derive the variant from the shared definition. |
| "Return the ORM record directly." | Return an explicit response model. |
| "Read `os.environ` / `process.env` where it is used." | Parse configuration once through a typed schema. |
