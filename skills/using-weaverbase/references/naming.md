# Name by Intent, Not by Type

Names should describe what something does or means, not the technical category
it belongs to. Generic type-suffixed names such as `Service`, `Manager`,
`Handler`, `Helper`, `Util`, `Data`, `Info`, and `State` describe the box, not
the intent.

## Rules

1. Drop generic type suffixes when a verb or domain noun is more precise.
2. A name should answer "what does this do?" or "what does this represent?",
   not "what kind of thing is this?".
3. If removing the suffix makes the name vague, the name was already vague.
   Rename the concept instead of re-adding the suffix.
4. Type-suffixed names are acceptable only when the suffix carries real
   technical meaning the reader needs (e.g. `UserRepository` vs
   `UserService` where repository is a deliberate pattern choice, or
   `AuthMiddleware` where `Middleware` denotes the framework contract).
5. Booleans and state describe the condition, not that they are state:
   `isReady`, `hasAccess`, not `readyState` or `accessFlag`.

## Examples

Good: names describe intent.

- `DocumentProcessor` (it processes documents)
- `useDocumentStatus` (exposes document status)
- `parseInvoice` (does the parsing)
- `BillingCycle` (a domain concept)
- `retryWithBackoff` (describes behavior)

Bad: names describe the type or container.

- `DocumentService` (service of what? does what?)
- `useDocumentServiceState` (leaks "service" and "state")
- `InvoiceHelper` / `InvoiceUtils` (does what to invoices?)
- `BillingManager` (manages how?)
- `RetryHandler` (handles retries by doing what?)

Good: the type suffix carries real meaning.

- `UserRepository` (deliberate Repository pattern)
- `AuthMiddleware` (framework contract)
- `OrderCreatedEvent` (event-sourcing concept)

Bad: renaming the concept is better than adding a weak suffix.

- `DataProcessor` over `DataService` is still weak. What data, what
  processing? Prefer `TranscriptNormalizer`, `PaymentReconciler`, etc.