# Avoid Using Python Protocol

Prefer explicit nominal contracts for internal Python code. Use `Protocol` only when a boundary is intentionally structural and readers do not need an explicit implementer relationship.

## Core Principle

`Protocol` answers "what shape can I consume?" It does not answer "who implements this?"

For internal contracts, readers often need to navigate from the contract to supported implementations. Parent/child inheritance and nominal base classes make that relationship easier to read, grep, and inspect with subclass navigation.

## Default for Internal Code

Avoid `Protocol` for internal service contracts, plugin contracts, adapters, and test seams when:

- Maintainers will ask "who implements this?"
- The project needs a central list of supported implementations.
- Implementers share construction rules, lifecycle hooks, registration, or orchestration policy.
- The framework or plugin system already requires inheritance, subclassing, or registration.
- The interface is large, unstable, or likely to accumulate methods.

Use an explicit base class or framework contract instead. Structural typing should not hide important implementation relationships.

## Use Protocol When

Use `Protocol` only when the boundary is truly about a small behavior that unrelated objects can satisfy structurally:

- External adapters should be swappable and callers only depend on behavior.
- Tests need lightweight fakes that implement the behavior under test.
- Multiple unrelated classes already share the same small capability.
- Consumer code needs only a narrow capability, not a broad implementation contract.
- Dependency inversion matters more than making implementers import a shared base class.

## Avoid Protocol When

Avoid `Protocol` when discoverability or nominal semantics are part of the contract:

- Developers frequently need to find every implementation.
- The project needs a supported-implementation manifest.
- Implementers must share construction rules or lifecycle hooks.
- A framework requires nominal inheritance or explicit registration.
- The interface is broad, unstable, or named after a vague role such as `Client` or `Provider`.

## Practical Exception Pattern

When `Protocol` is the right exception, keep it small and name it by capability:

```python
from dataclasses import dataclass
from typing import Protocol

@dataclass(frozen=True, slots=True)
class Message:
    recipient: str
    body: str


class SendsMessages(Protocol):
    async def send(self, message: Message) -> None:
        ...


async def notify(sender: SendsMessages, message: Message) -> None:
    await sender.send(message)
```

`notify` can use a production adapter, a test fake, or any other object with `send`. If future maintainers need a discoverable inventory of senders, solve that with an explicit registry, base class, or framework registration. Do not pretend the protocol provides implementation discovery.

## Decision Check

Before adding a protocol, answer both questions:

1. Will consumers benefit from accepting any object with this behavior?
2. Is it acceptable that implementers are not explicitly connected by inheritance?

If either answer is no, avoid `Protocol` for that boundary.

## Common Mistakes

- Using `Protocol` for internal contracts because it feels flexible, even though discoverability is the main requirement.
- Creating broad protocols named after nouns like `Client` or `Provider` instead of small capabilities like `SendsMessages` or `ReadsRecords`.
- Putting every method an implementation has into the protocol instead of only what the consumer needs.
- Treating a protocol as an implementation registry.
- Using a protocol to obscure a framework or plugin system's required contract.
