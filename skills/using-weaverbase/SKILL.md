---
name: using-weaverbase
description: Use when creating or modifying code, reviewing changes, or choosing architecture, validation, naming, Python contracts, Python logging, language, platform, database, Docker, or testing patterns
---

## Overview

Apply these rules before changing code, configuration, or review guidance.

**Core principle:** Read the relevant reference rule first, then treat it as a constraint. Urgency, compatibility habits, and "we can clean it up later" are not reasons to bypass the rule.

## When to Use

Use this skill for work involving:

- API, CLI, UI, worker, background job, or service boundaries
- Pydantic, Zod, request bodies, external responses, forms, env values, or message payloads
- Naming classes, hooks, functions, components, booleans, modules, or domain concepts
- Python, FastAPI, Python PDF processing, `Protocol`, internal contracts, adapters, plugin boundaries, test seams, TypeScript, JavaScript, React, Next.js, Node.js, or Rust code
- Dockerfiles, Docker Compose files, environment variables, labels, or Compose version syntax
- SQL schemas, database queries, ORM calls, migrations, or timestamp/JSON columns
- Configuration, logging, or test framework choices
- Code review for architecture, validation, naming, language, platform, database, Docker, or testing conventions

## Mandatory Reference Check

Before editing or giving implementation guidance, read every reference that applies:

| Situation | Required reference |
| --- | --- |
| Business logic, API routes, CLI commands, UI events, workers, jobs, service boundaries, domain errors | `references/service-first-architecture.md` |
| Pydantic, Pydantic model configuration, Zod, request/response shapes, form input, env parsing, external API payloads, `dict`, `unknown`, `any`, ad-hoc validation | `references/data-shapes-validation.md` |
| Names ending in `Service`, `Manager`, `Handler`, `Helper`, `Util`, `Data`, `Info`, `State`; unclear booleans or domain names | `references/naming.md` |
| Python `Protocol`, internal contracts, interfaces, adapters, plugin boundaries, test seams, structural typing, implementation discovery | `references/avoid-using-python-protocol.md` |
| Python type hints, dictionary merging, Python PDF libraries, FastAPI dependencies, async Python database access, Uvicorn logging config | `references/python-fastapi.md` |
| Python application logging defaults, bundled logging config template, app-local log config files, Uvicorn log format, `log_config`, `--log-config` | `references/python-logging.md`; also read `references/python-fastapi.md` for FastAPI/Uvicorn apps and `references/general-practices.md` for general logging rules |
| TypeScript, JavaScript, React, Next.js, ESM/CommonJS, async code, array transformations, routing, frontend state | `references/typescript-react-nextjs.md` |
| Node.js built-in imports, `node:` protocol, file operations, callback/sync filesystem APIs | `references/nodejs.md` |
| Rust error handling, `Result`, `unwrap`, string ownership, Tokio, async runtime choices | `references/rust.md` |
| Dockerfiles, base images, multi-stage builds, container users, `latest`, root containers | `references/docker.md` |
| SQL schemas, PostgreSQL JSON columns, timestamps, ORM/database queries in async apps | `references/sql-database.md` |
| Configuration, environment variables, logging, `print()`, test framework choices | `references/general-practices.md` |
| `docker-compose.yml`, `compose.yaml`, environment variables, labels, top-level `version`, YAML-sensitive values | `references/docker-compose.md` |

If more than one row applies, read all of them. If unsure, read the reference.

## Quick Rules

| Area | Do | Do not |
| --- | --- | --- |
| Architecture | Put business logic in a service layer that can be called from API, CLI, UI, and workers | Put workflows, database queries, subprocess calls, or domain decisions inside routes, commands, components, or job handlers |
| Function scope | Keep service and interface functions focused on one behavior | Mix orchestration, validation, formatting, persistence, and error mapping in one large function |
| Validation | Validate at boundaries with Pydantic models or Zod schemas; use Pydantic `model_config`; pass typed values inward | Accept raw `dict`, `unknown`, or `any` in services/components, re-check shape manually, or use nested Pydantic `Config` classes |
| Frontend data | Parse API/form payloads in route loaders, server actions, API-client adapters, form resolvers, or explicit boundary/container modules; pass typed props to presentational components | Put `safeParse`, `if (!value?.id)`, or render-null shape guards inside reusable or presentational components |
| Naming | Name by intent: what the thing does or represents | Use generic box names like `UserService`, `InvoiceHelper`, `BillingManager`, or `readyState` unless the suffix has real technical meaning |
| Python contracts | Prefer explicit nominal base classes or framework contracts for internal contracts where implementers must be discoverable | Use `Protocol` as an internal implementation registry, or hide framework/plugin contracts behind structural typing |
| Error responses | Map domain errors to safe user-facing messages at boundaries | Return raw exception text, stack traces, secrets, queries, file paths, payloads, or upstream internals |
| Python/FastAPI | Use `|` unions, dict `|` merges, `pypdfium2` for PDFs, async database drivers, and `Annotated[..., Depends(...)]` | Use `typing.Optional`, `typing.Union`, `{**a, **b}`, `pdf2image`, PyMuPDF/`fitz`, sync DB drivers in async apps, or default-parameter `Depends()` |
| TypeScript/React/Next.js | Use ESM, `const`/`let`, `async`/`await`, array methods, function components, hooks, and App Router for new work | Use CommonJS, `var`, promise chains by default, class components, or Pages Router APIs for new App Router work |
| Node.js | Use `node:` imports for built-ins and promise-based filesystem APIs | Import core modules without `node:` or use callback/sync filesystem APIs in application code |
| Rust | Use `Result`, `?`, borrowed `&str` where possible, and current Tokio async patterns | Use `.unwrap()`/`.expect()` in production or clone/allocate strings unnecessarily |
| Docker | Use specific base image tags, multi-stage builds, and non-root users | Use `latest`, ship build dependencies, or run production containers as root |
| SQL/database | Use PostgreSQL `JSONB`, `TIMESTAMPTZ`, and async ORM/database APIs in async apps | Use `JSON`, timezone-less timestamps, or blocking DB calls in async handlers |
| Configuration/logging | Validate environment configuration, use structured logging, and copy/apply `references/python-logging/log-config.json` into Python apps as an app-local config unless specified otherwise | Hardcode config, use `print()` for application logging, invent Python logging formats when the bundled template applies, or reference skill files at runtime |
| Tests | Add or update focused tests with `pytest`, `vitest`, or `jest` when behavior changes | Treat manual checks or urgency as enough, or start new Python projects with `unittest` by default |
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
- "Use `Protocol` for this internal contract because it feels flexible."
- "A protocol is enough to show who implements this contract."
- "Copy the old Compose `version: \"3.9\"` style because it still works."
- "Legacy reason" means a concrete tool/platform constraint, not personal familiarity or old examples.
- "Return a raw untyped dict/object first and add result types later."
- "Expose the raw exception detail so production is easier to debug."
- "Use `typing.Optional`, CommonJS, `var`, or promise chains because the old style is familiar."
- "Use nested Pydantic `Config` classes because older examples do it."
- "Use `pdf2image` or PyMuPDF (`fitz`) for Python PDF work because they are familiar."
- "Use Pages Router APIs for new Next.js App Router work."
- "Import Node built-ins without `node:` because it is shorter."
- "Use Docker `latest` or root containers for now and harden them later."
- "Use a synchronous database driver inside an async API or worker."
- "Use PostgreSQL `JSON` or `TIMESTAMP` because it is close enough."
- "Use `print()` for application logging or hardcode config for speed."
- "Create a new Python logging format instead of copying/applying `references/python-logging/log-config.json`, even though no other preference was specified."
- "Point app code, Dockerfiles, or Uvicorn startup commands at the skill file path instead of an app-local copied config."
- "Start a FastAPI app without passing the app-local copied logging config to Uvicorn, then fix logging after startup."
- "Start a new Python test suite with `unittest` by default."
- "This is a quick behavior change, so manual verification is enough."

## Rationalizations to Reject

| Rationalization | Response |
| --- | --- |
| "Since you’re in a hurry, I’ll keep this minimal and validate directly where it’s easy to see." | Boundary models are the minimal compliant path. Put structural validation on Pydantic/Zod and keep interfaces thin. |
| "We can replace the raw `dict` with a Pydantic model later once the endpoint shape settles." | The shape is the contract. Define it once now and derive variants when needed. |
| "I’ll use `UserService` for now since that’s the name you asked for." | Prefer an intent-based name unless the suffix communicates a deliberate technical contract. |
| "I’ll define this internal contract as a `Protocol` so implementations stay flexible." | Prefer explicit nominal base classes or framework contracts when readers need to find implementers. Use `Protocol` only for intentional narrow structural boundaries. |
| "Returning raw dicts or untyped objects is fine for a quick first pass." | Services return structured result types; interface serialization to JSON is a boundary concern. |
| "You asked for classic Compose syntax, so I kept `version` and env lists." | Use the current Compose Specification unless a named tool, platform, or legacy integration requires otherwise. Personal habit is not compatibility. |
| "No time to add schemas, so I’ll guard the component locally." | Parse external input at the boundary with Zod/Pydantic; components consume typed data. |
| "I used `safeParse` and returned `null`, so the component is safe." | `safeParse` belongs at the boundary. Presentational components receive typed props and do not decide whether external data is structurally valid. |
| "Return the raw exception detail so production debugging is easier." | Log internals through the appropriate channel; return safe user-facing messages from API, CLI, UI, and worker boundaries. |
| "I will set up Python logging later after the app starts." | Copy/apply `references/python-logging/log-config.json` into the app repository unless another preference is specified; FastAPI apps should pass that app-local config to Uvicorn so startup logs use the configured format. |
| "Tests will slow this down; I checked it manually." | Behavior changes need focused tests that prove the new behavior or regression fix. |
| "The legacy syntax still works, so it is fine." | Working code is not the same as current project convention. Use the modern language, framework, platform, and database practices unless a real constraint requires otherwise. |
| "The old Pydantic `Config` class is shorter and familiar." | Use Pydantic v2 `model_config` with `ConfigDict`, or `SettingsConfigDict` for settings models. |
| "The PDF library does not matter as long as it renders pages." | For Python PDF work, use `pypdfium2`; avoid `pdf2image` and PyMuPDF (`fitz`). |
| "This async endpoint only does one blocking call." | Blocking calls in async paths are still event-loop hazards. Use async drivers and async ORM/database APIs. |
| "The Dockerfile is only internal, so `latest` and root are fine." | Internal images still need reproducibility and least privilege. Use specific tags and non-root users. |

## Common Mistakes

- Reading only this file and skipping the references. This file routes you to the rule details; it does not replace them.
- Treating user-requested shortcuts as permission to violate project rules. Offer the compliant equivalent instead.
- Moving validation from the boundary into a service or component because the check is small.
- Using nested Pydantic `Config` classes instead of `model_config` with `ConfigDict` or `SettingsConfigDict`.
- Reaching for `pdf2image` or PyMuPDF (`fitz`) for Python PDF work instead of `pypdfium2`.
- Letting functions grow until they mix unrelated responsibilities instead of splitting by behavior and boundary concern.
- Calling an ordinary component a "boundary" so it can accept `unknown`/`any`, call `safeParse`, and hide invalid data by returning `null`; validate in a real boundary/container module and surface failures deliberately.
- Renaming `SomethingService` to `SomethingProcessor` without making the name more specific. If the concept is vague, rename the concept.
- Using `Protocol` for internal contracts where parent/child inheritance would make implementers easier to discover.
- Preserving obsolete Docker Compose syntax because it appears in old examples.
- Treating familiarity with old Compose syntax as a compatibility requirement.
- Keeping outdated language or framework style because it still runs, instead of checking the relevant topic reference.
- Using synchronous filesystem or database operations in async request handlers or workers.
- Adding new Dockerfiles with `latest` tags or root runtime users.
- Choosing weak SQL defaults such as `JSON` or timezone-less timestamps without a concrete compatibility reason.
- Using unvalidated environment values or `print()` logging in application code.
- Skipping the `references/python-logging/log-config.json` template for a Python app when the user or project has not specified another logging preference.
- Referencing `references/python-logging/log-config.json` from app runtime code, Dockerfiles, or deployment commands instead of copying/applying it into the app repository first.
- Letting FastAPI/Uvicorn start before applying the app-local copied logging config.
- Skipping focused tests for behavior changes because the change looks small or urgent.
- Returning raw exception details to users instead of safe boundary-level messages.
