# Name by intent, not by type

Names should answer *"what does this do?"* — not *"what kind of thing is this?"*

Generic type-suffixed names such as `Service`, `Manager`, `Handler`, `Helper`,
`Util`, `Data`, `Info`, and `State` describe the box, not the intent.

## Rules

1. **Drop generic type suffixes** when a verb or domain noun is more precise.

2. **If removing the suffix makes the name vague, the name was already vague.**
   Rename the concept instead of re-adding the suffix.

3. **Type suffixes are acceptable only when the suffix carries real technical
   meaning** the reader needs — a deliberate pattern choice (`UserRepository`),
   a framework contract (`AuthMiddleware`), or an architecture concept
   (`OrderCreatedEvent`).

4. **Booleans and conditions describe the state, not that they are state:**
   prefer `isReady`, `hasAccess` over `readyState`, `accessFlag`. The same
   applies to enums and constants: `PAYMENT_PENDING` over
   `PAYMENT_STATUS_CODE`.

5. **If you cannot find a precise name, that is a design signal.** A class that
   resists naming likely does too much. Consider splitting it before forcing a
   name.

## Examples

### Intent vs type

| Good — name describes intent | Bad — name describes the container |
|---|---|
| `DocumentProcessor` | `DocumentService` |
| `useDocumentStatus` | `useDocumentServiceState` |
| `parseInvoice` | `InvoiceHelper` / `InvoiceUtils` |
| `BillingCycle` | `BillingManager` |
| `retryWithBackoff` | `RetryHandler` |

### Suffixes that carry real meaning

```
# Good — suffix is the contract
UserRepository      # deliberate Repository pattern
AuthMiddleware      # framework hook
OrderCreatedEvent   # event-sourcing concept

# Bad — suffix is decoration only
UserService         # unless the framework enforces it as a DI concept
DataProcessor       # what data? what processing?
PaymentManager      # manages how?
```

Prefer `TranscriptNormalizer`, `PaymentReconciler` — name the domain concept,
not the category.

> **Framework exception.** If your framework treats a suffix as a registered
> contract — e.g. NestJS `@Injectable()` services, Spring `@Service` beans —
> the suffix is load-bearing. Apply the exception only when the framework
> *enforces* it, not merely when it is conventional.

## Quick test

Read the name aloud and ask: *"This does what, exactly?"*

If the honest answer is *"I'm not sure"*, the name is not doing its job.
