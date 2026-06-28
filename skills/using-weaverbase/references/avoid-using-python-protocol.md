# Avoid Using Python Protocol

Do not introduce `typing.Protocol` for application abstractions by default. Prefer explicit nominal contracts: concrete typed library classes, framework contracts, standard inheritance, or `abc.ABC`.

## Core Principle

`Protocol` answers "what shape can I consume?" It does not answer "who implements this?"

For internal Python code, readers usually need to navigate from a contract to its supported implementations. Parent/child inheritance and nominal base classes make that relationship easier to read, grep, inspect with subclass navigation, and maintain under strict typing.

## Default Rule

Avoid `Protocol` for internal service contracts, adapters, plugin contracts, and test seams.

Use one of these instead:

- A concrete typed external class when a library already exposes one.
- A framework base class or registration mechanism when the framework defines the contract.
- An `abc.ABC` when the application owns the contract and implementations must be discoverable.
- Standard inheritance when shared behavior or construction rules belong on a base class.
- A callable type alias when the dependency is only a function-shaped callback.

Structural typing should not hide important implementation relationships.

## Why This Matters

Avoiding internal `Protocol` contracts improves:

- IDE navigation from contract to implementation.
- Grep/search discoverability.
- Explicit ownership of application extension points.
- Clarity about which implementations are supported.
- Refactor safety when the contract changes.

## Avoid Protocol When

Avoid `Protocol` when any of these are true:

- Developers need to find every implementation.
- The project needs a supported-implementation manifest.
- The application owns the abstraction.
- The boundary is an internal service, adapter, plugin, or test seam.
- Implementers must share construction rules or lifecycle hooks.
- A framework requires nominal inheritance or explicit registration.
- The interface is broad, unstable, or named after a vague role such as `Client` or `Provider`.
- The only reason is making tests or fakes easier to write.

Tests can use mocks, fakes that inherit from the application ABC, or typed casts at the test boundary. Do not introduce `Protocol` solely for test convenience.

## Exception Gate

Do not add `typing.Protocol` unless a concrete constraint requires structural typing and an explicit design decision approves it.

Before using `Protocol`, confirm all of these:

1. A concrete external class, ABC, framework contract, inherited base class, or callable type is not sufficient.
2. Implementer discovery through inheritance is not required.
3. The code intentionally accepts unrelated objects by structural shape.
4. The protocol is small, capability-focused, and unlikely to accumulate methods.
5. The reason is documented near the boundary.

If any answer is uncertain, do not use `Protocol`.

## Decision Check

Before adding a contract, answer these questions:

1. Does a typed class or framework contract already exist?
2. Will maintainers need to navigate from the contract to its implementations?
3. Is this an application-owned dependency boundary?
4. Would an ABC or inherited base class communicate the relationship more clearly?

If any answer is yes, avoid `Protocol` for that boundary.

## Common Mistakes

- Using `Protocol` for internal contracts because it feels flexible, even though discoverability is the main requirement.
- Creating broad protocols named after nouns like `Client` or `Provider` instead of using an explicit application contract.
- Duplicating a typed external library contract with a local protocol.
- Using a protocol to make tests convenient instead of using mocks, ABC-backed fakes, or test-local casts.
- Treating a protocol as an implementation registry.
- Using a protocol to obscure a framework or plugin system's required contract.
