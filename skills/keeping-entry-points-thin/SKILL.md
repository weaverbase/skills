---
name: keeping-entry-points-thin
description: Use when implementing or changing an API route or handler, server action, CLI command, GUI event handler, queue consumer, scheduled job, or worker; when deciding whether logic belongs in the interface or the service layer; when mapping domain errors to HTTP status codes, exit codes, or UI messages; or when tempted to put SQL, a subprocess, or a workflow in a route because it is read-only, urgent, or one-off.
---

# Keeping Entry Points Thin

## Overview

Business logic lives in the service layer. API, CLI, GUI, and worker entry points are thin wires: **validate → call service → map output**.

**Litmus test:** if the logic should be callable unchanged from a CLI, an API, and a job, it belongs in the service layer.

These rules apply to new or requested changes. Do not refactor untouched legacy handlers solely to match them.

## Rules

1. **Business logic in the service layer.** Queries, decisions, workflows, and side effects live in services. Entry points never contain them, even when the logic is read-only, urgent, or one-off.
2. **Entry points are thin wires.** Validate the transport input (shape and format, with a boundary schema such as Pydantic or Zod), call one service operation, and map its result or error to the interface's output.
3. **Services take domain types and return typed results.** Never pass request, response, or context objects in, and never return raw dicts, untyped objects, or transport responses out. Serializing is the entry point's job.
4. **Services never raise transport exceptions.** A service must not raise `HTTPException`, call `sys.exit`, raise `typer.Exit`, or return a status code: those only make sense to one interface, and every other caller would break on them. Raise domain errors instead, such as `NotFoundError` or `ConflictError` subclassing a shared `DomainError` base, so each interface can map them to its own HTTP status, exit code, or UI message.
5. **Adding a new interface never changes a service.** A new API, CLI, GUI, or worker only adds a new thin wire. If a service must change to fit the new interface, the transport leaked into the service: fix the leak.

## Error Mapping

Map domain errors in one place per interface (an exception handler, a CLI wrapper, a job runner), not in every handler.

| Error | HTTP | CLI | GUI / worker |
| --- | --- | --- | --- |
| Input fails boundary validation | 400 or 422 | usage message, exit 2 | field message / reject message |
| `NotFoundError` | 404 | message on stderr, exit 1 | inline message / mark failed |
| `ConflictError` | 409 | message on stderr, exit 1 | inline message / mark failed |
| `PermissionDeniedError` | 403 | message on stderr, exit 1 | inline message / mark failed |
| Anything not a `DomainError` | log once, generic 500 | log once, generic message, exit 1 | log once, generic message / mark failed |

- Return safe messages only. Never expose stack traces, raw exception text, secrets, SQL, file paths, or upstream details.
- Never swallow an unexpected error into an empty or default success.

## Example

One service, two interfaces. The service knows nothing about HTTP or the CLI; each interface validates, calls, and maps.

```python
from dataclasses import dataclass
from uuid import UUID

import typer
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from pydantic import BaseModel, EmailStr, Field


# Service layer
class DomainError(Exception):
    pass


class NotFoundError(DomainError):
    pass


class ConflictError(DomainError):
    pass


@dataclass(frozen=True)
class User:
    id: UUID
    email: str
    name: str


class Users:
    def register(self, email: str, name: str) -> User:
        ...  # raises ConflictError when the email is taken


# API: validate -> call service -> map output
class CreateUser(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


app = FastAPI()
users = Users()
HTTP_STATUS = {NotFoundError: 404, ConflictError: 409}


@app.exception_handler(DomainError)
async def domain_error(_: Request, exc: DomainError) -> JSONResponse:
    return JSONResponse(status_code=HTTP_STATUS.get(type(exc), 400), content={"error": str(exc)})


@app.post("/users", status_code=201)
def create_user(body: CreateUser) -> User:
    return users.register(body.email, body.name)


# CLI: same service, unchanged
cli = typer.Typer()


@cli.command()
def register(email: str, name: str) -> None:
    try:
        user = users.register(email, name)
    except DomainError as exc:
        typer.echo(str(exc), err=True)
        raise typer.Exit(1) from exc
    typer.echo(user.id)
```

Domain error messages are written for users (`"email already registered"`), so `str(exc)` is safe for `DomainError`. Never return `str(exc)` for other exceptions.

Bad: the route owns the workflow, so a CLI or job cannot reuse it without copying.

```python
@app.post("/deploy/{stack}")
def deploy(stack: str):
    subprocess.run(["docker", "compose", "up", "-d"], cwd=f"/stacks/{stack}")
    return {"status": "ok"}
```

Fix: move it to `stacks.deploy(stack)`, which returns a typed result and raises `NotFoundError` for an unknown stack. The route only calls and maps.

Preserve the roles, not these types or file layout, in other stacks. In Go, use a shared base error type or sentinel matched with `errors.As`/`errors.Is`; in TypeScript, `class NotFoundError extends DomainError`.

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "It is only a SELECT, so SQL in the route is fine." | Read-only logic is still business logic. Put it in the service layer. |
| "This is urgent, so keep it in the handler." | A thin wire calling one service method is the fastest compliant path. |
| "The CLI needs it slightly differently, so copy the logic into the command." | Call the same service. Adapt only input and output. |
| "Return a dict or status code from the service to shorten the handler." | Return a typed result or raise a `DomainError`. The interface maps it. |
| "Raise `HTTPException(404)` in the service; the API needs a 404 anyway." | The CLI and jobs cannot handle it. Raise `NotFoundError`; the API maps it to 404. |
| "Change the service signature so the new interface fits." | A new interface never changes a service. Adapt in the wire. |

## Red Flags

- SQL, an ORM call, or `subprocess` in a route, command, GUI handler, or job handler.
- A service signature with a request, response, or context type, or a service returning a status code or raw dict.
- A service importing or raising a transport exception (`HTTPException`, `typer.Exit`, `sys.exit`).
- `str(exc)` of a non-domain exception in a response.
- The same logic in two entry points.
