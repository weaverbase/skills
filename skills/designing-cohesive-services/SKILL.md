---
name: designing-cohesive-services
description: Use when adding or changing business logic in the service layer; when deciding whether logic belongs in a service method, standalone function, query function, or helper; when a service is becoming a pass-through wrapper, a class per action, or a catch-all; or when someone proposes a repository, use-case, or interface layer.
---

# Designing Cohesive Services

## Overview

Business logic lives in the service layer, as methods on cohesive services or as standalone functions. This skill is about how to shape and group that layer.

**Core principle:** Group operations in a service because they share state, a resource, or an invariant. "Service layer" does not mean every function needs a service wrapper, and it does not mean extra layers beneath it.

These preferences apply to new or requested changes. Do not refactor, rename, or reorganize existing working code solely to match them. Explicit task instructions and the repository's own guides outrank these defaults.

## Service Contract in Brief

- Callable unchanged from any API, CLI, GUI, or worker; never takes request, response, or context objects.
- Takes domain types and returns typed results, never raw dicts or untyped objects.
- Raises domain errors (for example `NotFoundError` subclassing a shared `DomainError`), never transport exceptions or status codes.

## Quick Reference

| Observable need | Shape |
| --- | --- |
| Operations share state, resource lifecycle, or invariants | Methods on the same concrete service. |
| Behavior depends only on its inputs and shares no service responsibility | Standalone function; callers use it directly. |
| Complex read deserves a distinct home | Concrete query function or module. |
| Decision is clearer or reused when separated | Pure helper called by the owning operation. Extraction is optional. |
| A real implementation boundary varies or must be isolated | Small interface or port at that boundary, not an interface for every class. |
| Operation produces several values | Readable multiple typed returns (`order, receipt, err`) or a typed result (`OrderSummary`). |
| A function keeps growing | Split by behavior, not into architectural layers. |

## Rules

- **Group by shared responsibility.** Methods belong on one service when they share state, a resource, or an invariant. A service whose methods share none of these is a catch-all; split it. Several classes that repeat the same lookup or client are one service; merge them.
- **No pass-through wrappers.** A method whose body only forwards to another function adds nothing. Call the function directly.
- **One method is fine.** A single-method service is fine when it is the cohesive home of its behavior or owns real state. Do not invent methods to reach a count.
- **No mandatory layers.** Do not add a repository, use-case, or functional-core layer, or an interface per class, for symmetry. Extract a helper or abstraction only for a demonstrated reason, such as a real implementation boundary or reuse.
- **Split by behavior.** When a function mixes responsibilities, split along behavior lines (place the order, build the receipt, query the history), not along layers.
- **Name methods for actions.** `orders.place`, `orders.cancel`, `stacks.deploy`. The method is the operation itself, not a wrapper around another class.

## Example

A cohesive service: deploy, stop, and status share the stack directory lookup, its path validation, and the compose client.

```python
from dataclasses import dataclass
from pathlib import Path


class DomainError(Exception):
    pass


class NotFoundError(DomainError):
    pass


@dataclass(frozen=True)
class DeployResult:
    stack: str
    services_started: int


@dataclass(frozen=True)
class StackStatus:
    stack: str
    running: bool


class Stacks:
    """Deploy, stop, and inspect stacks that live under one root directory."""

    def __init__(self, root: Path, compose: ComposeClient) -> None:
        # ComposeClient is a concrete wrapper around the compose CLI.
        self._root = root.resolve()
        self._compose = compose

    def deploy(self, stack: str) -> DeployResult:
        started = self._compose.up(self._directory(stack))
        return DeployResult(stack=stack, services_started=started)

    def stop(self, stack: str) -> None:
        self._compose.down(self._directory(stack))

    def status(self, stack: str) -> StackStatus:
        running = self._compose.is_running(self._directory(stack))
        return StackStatus(stack=stack, running=running)

    def _directory(self, stack: str) -> Path:
        directory = (self._root / stack).resolve()
        if directory.parent != self._root or not directory.is_dir():
            raise NotFoundError(f"stack {stack!r} not found")
        return directory
```

An input-only calculation needs no service state, so it stays a standalone function instead of an `OrderService.shippingBand` delegate:

```typescript
export function shippingBand(weightGrams: number): "standard" | "heavy" {
  if (!Number.isSafeInteger(weightGrams) || weightGrams < 0) {
    throw new RangeError("weight must be nonnegative whole grams");
  }
  return weightGrams < 2_000 ? "standard" : "heavy";
}
```

Bad: one class per action, each a pass-through that repeats the same lookup and shares the same client. The shape is wrong because the responsibilities are shared, not because each class has one method.

```python
class StackDeployer:
    def deploy(self, stack: str) -> DeployResult:
        return self._compose.up(self._root / stack)


class StackStopper:
    def stop(self, stack: str) -> None:
        self._compose.down(self._root / stack)  # lookup and validation duplicated or missing
```

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "All logic must be a service method." | An independent calculation does not belong on an unrelated service. Use a standalone function. |
| "Make one service per method, or put everything in one big service." | Group by shared state, resource, or invariant. |
| "A service with one method must be wrong, so add more." | Fine when it is the cohesive home of its behavior. |
| "Add a repository layer, use-case class, or interface for symmetry." | Only for a demonstrated reason. Do not import another language's class hierarchy or directory layout. |
| "Wrap the existing function in a service method for consistency." | A pass-through adds nothing. Call the function. |
| "Let the function grow; it all happens together anyway." | Split by behavior once it mixes responsibilities. |

## Red Flags

- A service method whose body only forwards to another method or function.
- A service whose methods share no state, resource, or invariant.
- Several single-action classes repeating the same lookup or client.
- An interface with exactly one implementation and no real boundary behind it.
- A new layer added "for consistency" with no behavior of its own.
