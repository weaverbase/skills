# Service-First Architecture

Business logic belongs in a service layer. UI, CLI, API routes, and background
jobs should be thin wires that validate input, call the service, and map the
result to their interface format.

## Litmus Test

Before writing logic, ask:

> Can this be called from a CLI, an API route, a UI event, and a background job
> without changing the implementation?

- Yes: put it in the service layer.
- No: keep it in the interface layer.

## Rules

1. Write the service first. Define what the system does before deciding how an
   interface calls it.
2. Keep interfaces thin. Each interface should validate input, call the service,
   and map the output.
3. Keep service and interface functions focused on one behavior. Split
   orchestration, validation, formatting, persistence, and error mapping when a
   function starts mixing responsibilities.
4. Never put business logic in interface handlers. Avoid `subprocess`, queries,
   and domain workflows in API routes, CLI commands, or UI handlers.
5. Service methods use plain domain types. Do not use `Request`, `Response`, or
   framework-specific types in service signatures.
6. Return structured result types from services. Do not return raw dicts,
   untyped objects, or framework responses.
7. Map errors at the boundary. Services raise domain errors; interfaces translate
   them into HTTP status codes, exit codes, UI messages, or job failure states.
8. Adding a new interface should not require changing the service. If it does,
   the service boundary is not clean enough.

## Examples

Good: the API delegates to reusable business logic.

```python
class StackDeployer:
    def deploy(self, stack: str) -> DeployResult:
        ...


@router.post("/deploy/{stack}")
def deploy(stack: str, deployer: Annotated[StackDeployer, Depends(get_deployer)]):
    return deployer.deploy(stack)
```

Bad: the API route owns the workflow.

```python
@router.post("/deploy/{stack}")
def deploy(stack: str):
    subprocess.run(["docker", "compose", "pull"], cwd=f"/stacks/{stack}")
    subprocess.run(["docker", "compose", "up", "-d"], cwd=f"/stacks/{stack}")
    return {"status": "ok"}
```

## Error Mapping

Keep domain errors in the service layer and translate them at each boundary.
Return safe user-facing messages; do not expose stack traces, raw exception text,
secrets, SQL, internal payloads, file paths, or upstream service details.

```python
class StackNotFoundError(Exception):
    pass


@router.post("/deploy/{stack}")
def deploy(stack: str, deployer: Annotated[StackDeployer, Depends(get_deployer)]):
    try:
        return deployer.deploy(stack)
    except StackNotFoundError as exc:
        raise HTTPException(status_code=404, detail="Stack not found") from exc
```
