---
name: handling-transactions-and-side-effects
description: Use when writing or changing code that commits or rolls back a database transaction, retries jobs or requests, sends emails or webhooks, publishes events, calls external APIs that write state, runs multi-step workflows that can fail partway, or reconciles stored intent with live external state; also for outbox, idempotency, race condition, duplicate delivery, partial failure, concurrency limit, and cancellation questions.
---

# Handling Transactions and Side Effects

## Overview

An operation that changes state decides where the transaction starts and ends, and when anything outside the database may happen. Getting this wrong produces duplicate emails, lost events, lost updates, and half-applied changes that look healthy.

**Core principle:** Commit first, then affect the outside world, and make automatic retries safe. Never assume a step either fully happened or fully did not.

These rules apply to new or requested changes. Do not rewrite existing working code solely to match them.

Match the protection to the realistic risk. Guard races and edge cases that the code's actual usage makes plausible and that would cause real harm (lost money or stock, duplicate records that break invariants, wrong access). Do not add locks, isolation changes, conflict handling, or extra branches for hypothetical interleavings in code that is not concurrently written, or where a rare wrong outcome is cheap and visible. When you notice such a risk but leave it unguarded, mention it in the report instead of fixing it unasked.

## Quick Reference

| Situation | Rule |
| --- | --- |
| Who commits? | The owning operation defines the transaction boundary. Persistence helpers take the transaction or session and never commit independently. |
| Check-then-write on data that concurrent requests realistically contend for, where a wrong outcome is costly (stock, balance, uniqueness, access grants) | Keep the check and change in one transaction, protected by a conditional write, appropriate row locks, a constraint, or isolation that prevents the race. A transaction alone is not enough. |
| Race that is unlikely in practice, or whose wrong outcome is cheap and visible (single-user tool, admin script, last-write-wins field) | Do not add protection. Rely on existing constraints; note the risk in the report if it seems worth a follow-up. |
| Transaction length | Keep transactions short. No network or other external calls inside them. |
| Email, webhook, event, or any externally visible effect | Perform it only after commit. When delivery must be atomic with the write, commit an outbox row (or an existing equivalent) with the write and deliver from it afterward. |
| Code that retries automatically (retried job, redelivered message, retry loop, outbox dispatcher) | Guard with an idempotency key, unique constraint, or recoverable state. Do not assume an automatic retry is harmless. |
| Failure surfaced to the user as a handled error | Retry policy belongs to the project. Return the error; do not add idempotency machinery unless the project asks for it. |
| Fan-out or parallel work the code actually performs | Bound the concurrency, propagate cancellation and timeouts, and keep shared state race-safe. |
| Error inside a transaction | Let it roll back and propagate. Do not catch, continue, and commit a half-done change. |
| Multi-step change to an external system | See "External Workflows and Reconciliation". |

## Rules

### Transaction boundary

The operation that owns the behavior opens, commits, and rolls back the transaction. Repositories, query helpers, and other persistence helpers accept the transaction or session as a parameter. A helper that commits on its own makes the enclosing operation impossible to make atomic.

### Races

Protect a decision together with the change it guards when concurrent requests realistically contend for the same data and a wrong outcome would cause real harm: overselling stock, overdrawing a balance, a duplicate that breaks a uniqueness invariant, or granting access that should be denied. Skip this for data that only one actor writes at a time, for low-stakes fields where last write wins is acceptable, and for scripts and tools that do not run concurrently. Merely putting a read and write inside one transaction, especially at READ COMMITTED isolation, does not prevent another writer from invalidating the check. Use a conditional write, appropriate row locks, a relevant constraint, or an isolation level that prevents the anomaly; handle conflicts and serialization failures. Prefer a conditional write or a unique constraint over unprotected read-then-write, and check the affected-row count rather than assuming success. A check done earlier in middleware or in an earlier query is a hint, not a guarantee.

### Short transactions, no external calls

Do preparation and slow work before opening the transaction. Never make a network call, call an external API, or wait on another system while holding one: it holds locks, and its failure or latency decides whether the database work commits.

### Effects after commit

Send emails, call webhooks, publish events, and enqueue jobs only after the commit succeeded; otherwise the outside world sees a change that rolled back. When losing the effect is unacceptable, write an outbox row (a record of the intended effect) in the same transaction, then deliver it from a separate dispatcher that marks it sent after the effect succeeds. Use the project's existing equivalent if it has one. Delivery is then at least once, so the receiver or the dispatcher must tolerate duplicates.

### Retries and partial failure

When the system itself repeats work (retried jobs, redelivered messages, retry loops, an at-least-once dispatcher, a resumed workflow after a crash), make the second run harmless with:

- an idempotency key sent to the external service and stored with the record,
- a unique constraint that turns a duplicate into a detectable conflict,
- recoverable state (`pending`, `sent`, `failed`) that a later run can resume from.

Test the automatic retry and crash-between-steps paths, not only the happy path.

When a failure is returned to the user as a handled error and the user decides whether to try again, that retry policy is the project's choice. Return a clear error and leave duplicate protection for user-initiated retries to the project's requirements; do not add idempotency keys, request deduplication, or retry logic that was not asked for.

### Concurrency and cancellation

When the code fans out over many items, use bounded concurrency (a worker pool, semaphore, or batch size) instead of one task per item. Pass cancellation and timeouts through to external calls so a cancelled request stops work. When in-memory state is actually shared across concurrent tasks or threads, protect it with the language's race-safe tools (locks, atomics, single-owner channels) rather than relying on timing. Sequential code needs none of this.

## Example

Place an order: stock is reserved with a conditional write, the order and an outbox row commit together, and the email is sent later by a dispatcher with an idempotency key.

```typescript
async function placeOrder(db: Database, input: PlaceOrder): Promise<Order> {
  return db.transaction(async (tx) => {
    // Conditional write: affects 0 rows when stock is insufficient.
    const reserved = await tx.reserveStock(input.sku, input.quantity);
    if (!reserved) throw new OutOfStock(input.sku);

    const order = await tx.insertOrder(input);
    await tx.insertOutbox({ key: `order-placed:${order.id}`, topic: "order.placed", orderId: order.id });
    return order; // commit happens when the callback returns
  });
}

// Separate dispatcher, runs after commit and is safe to repeat.
async function dispatchOutbox(db: Database, mailer: Mailer): Promise<void> {
  for (const row of await db.claimUnsent(50)) {
    await mailer.sendOrderConfirmation(row.orderId, { idempotencyKey: row.key });
    await db.markSent(row.id);
  }
}
```

Bad: a network call inside the transaction, a helper that commits itself, and an effect that can outlive a rollback.

```typescript
await db.transaction(async (tx) => {
  const order = await tx.insertOrder(input);
  await mailer.sendOrderConfirmation(order.id);   // email goes out even if the commit fails
  await tx.reserveStock(input.sku, input.quantity); // can fail after the email was sent
  await orderRepository.save(order);                // helper opens its own transaction and commits
});
```

## External Workflows and Reconciliation

Apply this section when code changes state in an external system (a gateway, proxy, DNS, cloud resource, identity provider, or provider API) in several steps, or through writes whose failure mode is unclear. A single idempotent call with clear success and failure semantics needs only the retry and idempotency rules above.

| If this is true | Then |
| --- | --- |
| The change takes several external steps, or the process can crash between them | Persist the intent (desired state, operation id, steps completed) before the first external step, so a later run can resume or reconcile. Verify the final desired state with a matching read-back and a functional probe through the real path before finalizing; intermediate step successes alone are not proof of the final outcome. |
| A failed or ambiguous step can leave partial state that is unsafe (live credentials, open access, a half-applied route) | Reconcile toward the safe state defined for that workflow. A failed create or grant cleans up toward absence, removing only what this operation recorded creating. A failed delete, disable, or revoke keeps retrying toward revocation. |
| Leftovers are harmless, may hold data, or may have existed before this operation | Do not delete them. Retry idempotently from the recorded intent, or flag the workflow for review. |
| A write replaces the complete state (whole-document PUT, apply-all, replace-list), or a non-success response (error, timeout, 5xx) may still have changed live state | Treat the outcome as unknown. Do not finalize until the live result is verified: a fresh successful write, a read-back that matches the intent, and a functional probe through the real path. A liveness check (a `/healthz` 200) or a management API 200 is not proof of readiness. |
| Several writers touch the same external state, especially with read-modify-write or complete-state writes | Serialize them: one writer, a lock, a queue, or a compare-and-swap on a version or ETag. |
| Opening traffic, routes, listeners, or access depends on that external state being correct | Keep the exposure closed until the live state is verified; open it last. |
| Always | Never reuse a privileged management credential as a client credential or as a fallback when the intended credential fails. Use a separate, least-privilege credential per role. |

When none of the conditions apply, a checked success response from the API's documented contract is enough verification. Do not add destructive cleanup or extra probes to workflows that do not need them.

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "The helper can just commit so the caller does not have to." | The owning operation defines the transaction. Helpers join it. |
| "I checked the balance earlier, so the write is safe." | Protect the check and write with a conditional write, lock, constraint, or adequate isolation; merely sharing a transaction is not enough. |
| "A quick HTTP call inside the transaction is fine." | External calls go outside. Slow or failing calls must not decide whether the commit holds locks or succeeds. |
| "Send the email now; the commit will almost certainly succeed." | Send after commit, or write an outbox row with the change. |
| "The job retries automatically, so it is resilient." | A retry without an idempotency key, constraint, or recoverable state duplicates the effect. |
| "The management API returned 200, so the new config is live." | A 200 or a liveness check is not readiness. Read back and probe the real path when writes replace complete state or failures may still mutate it. |
| "The request errored, so nothing changed." | A non-success write may have partially applied. Reconcile or verify before treating the state as unchanged. |
| "Clean up by deleting whatever looks leftover." | Delete only what this operation recorded creating, and only where absence is the safe state. |
| "Use the admin credential as a fallback so the client keeps working." | Never reuse a privileged credential as a client or fallback credential. |
| "Spawn a task per item; the runtime will cope." | Bound concurrency and propagate cancellation. |
| "Two requests could theoretically interleave here, so add a lock and conflict handling." | Only if concurrent writes are realistic and the wrong outcome is costly. Otherwise leave the code simple and mention the risk in the report if it matters. |

## Red Flags

Stop and reconsider if you see:

- `commit()` inside a repository or helper.
- `SELECT` followed by `UPDATE` on contended, high-stakes data (stock, balance, uniqueness, access) with no conditional write, appropriate lock, relevant constraint, or adequate isolation, even when they share a transaction.
- `fetch`, `requests`, an SDK call, or a mailer inside a transaction callback.
- An effect scheduled before the commit has succeeded.
- A retry loop around a non-idempotent call.
- A catch block inside a transaction that continues and commits.
- Finalizing an external change based only on a 200 or a health check when the write replaces complete state or failures may still mutate it.
- A privileged credential used as a client or fallback credential.
