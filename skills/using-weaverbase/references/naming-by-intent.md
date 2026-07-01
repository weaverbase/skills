# Name class and type concepts by intent, not by category

Class and type names should describe what the concept represents or is responsible
for, not the generic technical category it belongs to.

Generic type-suffixed class/type names such as `Service`, `Manager`, `Handler`,
`Helper`, `Util`, `Data`, `Info`, and `State` describe the box, not the intent.
This rule applies only to class and type names. Do not use it to police function,
hook, module, variable, property, boolean, enum, or constant names.

## Rules

1. **Apply this rule only to class and type names.** Functions, hooks, modules,
   properties, booleans, enums, and constants are outside this reference's
   scope.

2. **Drop generic type suffixes** when a precise domain noun or responsibility
   name communicates the class/type's intent better.

3. **If removing the suffix makes the name vague, the concept was already
   vague.** Rename the class/type concept instead of re-adding the suffix.

4. **Type suffixes are acceptable only when the suffix carries real technical
   meaning** the reader needs — a deliberate pattern choice (`UserRepository`),
   a framework contract (`AuthMiddleware`), or an architecture concept
   (`OrderCreatedEvent`).

5. **If you cannot find a precise class/type name, that is a design signal.** A
   class or type that resists naming may be doing too much. Consider splitting it
   before forcing a vague name.

## Examples

### Intent vs category

| Good — class/type name describes intent | Bad — class/type name describes the container |
|---|---|
| `UserRegistrar` | `UserService` |
| `InvoiceParser` | `InvoiceHelper` / `InvoiceUtils` |
| `PaymentReconciler` | `PaymentManager` |
| `TranscriptNormalizer` | `DataProcessor` |
| `DocumentStatusSnapshot` | `DocumentState` |

### Suffixes that carry real meaning

```
# Good — suffix is the contract
UserRepository      # deliberate Repository pattern
AuthMiddleware      # framework contract
OrderCreatedEvent   # event-sourcing concept

# Bad — suffix is decoration only
UserService         # unless the framework enforces it as a DI concept
DataProcessor       # what data? what processing?
PaymentManager      # manages how?
```

Prefer `TranscriptNormalizer`, `PaymentReconciler`, and `UserRegistrar` — name
the class/type concept, not the category.

> **Framework exception.** If your framework treats a suffix as a registered
> contract — e.g. NestJS `@Injectable()` services, Spring `@Service` beans —
> the suffix is load-bearing. Apply the exception only when the framework
> *enforces* it, not merely when it is conventional.

## Quick test

For class/type names, read the name aloud and ask: *"This represents what,
exactly?"*

If the honest answer is *"I'm not sure"*, the class/type name is not doing its
job.
