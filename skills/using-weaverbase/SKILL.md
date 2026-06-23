---
name: using-weaverbase
description: Use when creating or modifying code, reviewing changes, or choosing architecture, validation, naming, or Docker Compose patterns
---

## Overview

Apply these rules before changing code, configuration, or review guidance.

**Core principle:** Read the relevant reference rule first, then treat it as a constraint. Urgency, compatibility habits, and "we can clean it up later" are not reasons to bypass the rule.

## When to Use

Use this skill for work involving:

- API, CLI, UI, worker, background job, or service boundaries
- Pydantic, Zod, request bodies, external responses, forms, env values, or message payloads
- Naming classes, hooks, functions, components, booleans, modules, or domain concepts
- Docker Compose files, environment variables, labels, or Compose version syntax
- Code review for architecture, validation, naming, or Compose conventions

## Mandatory Reference Check

Before editing or giving implementation guidance, read every reference that applies:

| Situation | Required reference |
| --- | --- |
| Business logic, API routes, CLI commands, UI events, workers, jobs, service boundaries, domain errors | `references/service-first-architecture.md` |
| Pydantic, Zod, request/response shapes, form input, env parsing, external API payloads, `dict`, `unknown`, `any`, ad-hoc validation | `references/data-shapes-validation.md` |
| Names ending in `Service`, `Manager`, `Handler`, `Helper`, `Util`, `Data`, `Info`, `State`; unclear booleans or domain names | `references/naming.md` |
| `docker-compose.yml`, `compose.yaml`, environment variables, labels, top-level `version`, YAML-sensitive values | `references/docker-compose.md` |

If more than one row applies, read all of them. If unsure, read the reference.

## Quick Rules

| Area | Do | Do not |
| --- | --- | --- |
| Architecture | Put business logic in a service layer that can be called from API, CLI, UI, and workers | Put workflows, database queries, subprocess calls, or domain decisions inside routes, commands, components, or job handlers |
| Function scope | Keep service and interface functions focused on one behavior | Mix orchestration, validation, formatting, persistence, and error mapping in one large function |
| Validation | Validate at boundaries with Pydantic models or Zod schemas; pass typed values inward | Accept raw `dict`, `unknown`, or `any` in services/components and re-check shape manually |
| Frontend data | Parse API/form payloads in route loaders, server actions, API-client adapters, form resolvers, or explicit boundary/container modules; pass typed props to presentational components | Put `safeParse`, `if (!value?.id)`, or render-null shape guards inside reusable or presentational components |
| Naming | Name by intent: what the thing does or represents | Use generic box names like `UserService`, `InvoiceHelper`, `BillingManager`, or `readyState` unless the suffix has real technical meaning |
| Error responses | Map domain errors to safe user-facing messages at boundaries | Return raw exception text, stack traces, secrets, queries, file paths, payloads, or upstream internals |
| Tests | Add or update focused tests when behavior changes | Treat manual checks or urgency as enough for changed behavior |
| Compose | Omit top-level `version`; use mapping syntax for `environment` and `labels`; quote YAML-sensitive values | Copy obsolete Compose examples with `version: "3.9"` and `KEY=value` lists by default |

## One Good Example

A FastAPI endpoint validates at the boundary, delegates to reusable business logic, uses intent-based names, and maps domain errors in the interface layer:

```python
from typing import Annotated
from uuid import UUID

from fastapi import APIRouter, Depends, HTTPException
from pydantic import BaseModel, ConfigDict, EmailStr, Field

router = APIRouter()


class CreateUser(BaseModel):
    model_config = ConfigDict(frozen=True)

    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


class CreatedUser(BaseModel):
    model_config = ConfigDict(frozen=True)

    id: UUID
    email: EmailStr
    name: str


class DuplicateUserError(Exception):
    pass


class UserRegistrar:
    def register(self, user: CreateUser) -> CreatedUser:
        ...


def get_user_registrar() -> UserRegistrar:
    return UserRegistrar()


@router.post("/users", response_model=CreatedUser)
def create_user(
    user: CreateUser,
    registrar: Annotated[UserRegistrar, Depends(get_user_registrar)],
) -> CreatedUser:
    try:
        return registrar.register(user)
    except DuplicateUserError as exc:
        raise HTTPException(status_code=409, detail="User already exists") from exc
```

## Red Flags

Stop and read the applicable reference before continuing if you think or see:

- "This is urgent, so keep it directly in the route/component/command."
- "We can replace the raw `dict`/`any` with a schema later."
- "A quick `if (!value?.id)` check is enough for now."
- "Use `Service`, `Helper`, or `Manager` because it is familiar."
- "Copy the old Compose `version: \"3.9\"` style because it still works."
- "Legacy reason" means a concrete tool/platform constraint, not personal familiarity or old examples.
- "Return a raw untyped dict/object first and add result types later."
- "Expose the raw exception detail so production is easier to debug."
- "This is a quick behavior change, so manual verification is enough."

## Rationalizations to Reject

| Rationalization | Response |
| --- | --- |
| "Since you’re in a hurry, I’ll keep this minimal and validate directly where it’s easy to see." | Boundary models are the minimal compliant path. Put structural validation on Pydantic/Zod and keep interfaces thin. |
| "We can replace the raw `dict` with a Pydantic model later once the endpoint shape settles." | The shape is the contract. Define it once now and derive variants when needed. |
| "I’ll use `UserService` for now since that’s the name you asked for." | Prefer an intent-based name unless the suffix communicates a deliberate technical contract. |
| "Returning raw dicts or untyped objects is fine for a quick first pass." | Services return structured result types; interface serialization to JSON is a boundary concern. |
| "You asked for classic Compose syntax, so I kept `version` and env lists." | Use the current Compose Specification unless a named tool, platform, or legacy integration requires otherwise. Personal habit is not compatibility. |
| "No time to add schemas, so I’ll guard the component locally." | Parse external input at the boundary with Zod/Pydantic; components consume typed data. |
| "I used `safeParse` and returned `null`, so the component is safe." | `safeParse` belongs at the boundary. Presentational components receive typed props and do not decide whether external data is structurally valid. |
| "Return the raw exception detail so production debugging is easier." | Log internals through the appropriate channel; return safe user-facing messages from API, CLI, UI, and worker boundaries. |
| "Tests will slow this down; I checked it manually." | Behavior changes need focused tests that prove the new behavior or regression fix. |

## Common Mistakes

- Reading only this file and skipping the references. This file routes you to the rule details; it does not replace them.
- Treating user-requested shortcuts as permission to violate project rules. Offer the compliant equivalent instead.
- Moving validation from the boundary into a service or component because the check is small.
- Letting functions grow until they mix unrelated responsibilities instead of splitting by behavior and boundary concern.
- Calling an ordinary component a "boundary" so it can accept `unknown`/`any`, call `safeParse`, and hide invalid data by returning `null`; validate in a real boundary/container module and surface failures deliberately.
- Renaming `SomethingService` to `SomethingProcessor` without making the name more specific. If the concept is vague, rename the concept.
- Preserving obsolete Docker Compose syntax because it appears in old examples.
- Treating familiarity with old Compose syntax as a compatibility requirement.
- Skipping focused tests for behavior changes because the change looks small or urgent.
- Returning raw exception details to users instead of safe boundary-level messages.
