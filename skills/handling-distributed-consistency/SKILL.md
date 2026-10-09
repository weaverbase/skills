---
name: handling-distributed-consistency
description: Use when an effect must reliably happen after a database write (outbox, at-least-once delivery), when code retries automatically (job queues, redelivered messages, retry loops), when concurrent requests contend for the same data (stock, balance, uniqueness, access grants), when work fans out in parallel, or when a workflow changes state in an external system in several steps and must reconcile partial failure; also for idempotency, duplicate delivery, race condition, and reconciliation questions.
---

# Handling Distributed Consistency

## Overview

Advanced patterns for keeping a database and the outside world consistent when work is retried, delivered more than once, raced, or applied in several external steps. Ordinary CRUD apps rarely need them.

**Core principle:** Match the protection to the realistic risk. Never assume a step either fully happened or fully did not, but only add machinery where the code's actual usage makes the failure plausible and its outcome costly.

These rules apply to new or requested changes. Do not rewrite existing working code solely to match them. Do not add locks, isolation changes, conflict handling, idempotency keys, or reconciliation for hypothetical failures. When you notice a real risk but leave it unguarded, mention it in the report instead of fixing it unasked.

## Quick Reference

| Situation | Rule |
| --- | --- |
| An effect must not be lost when the write commits | Commit an outbox row with the write; deliver it afterward from a dispatcher. |
| Code retries automatically (retried job, redelivered message, retry loop, outbox dispatcher) | Guard with an idempotency key, unique constraint, or recoverable state. |
| Failure surfaced to the user as a handled error | Retry policy belongs to the project. Return the error; add no idempotency machinery unless asked. |
| Check-then-write on contended, costly data (stock, balance, uniqueness, access grants) | Protect the check and write together with a conditional write, row lock, constraint, or adequate isolation. |
| Race that is unlikely, or whose wrong outcome is cheap and visible | Add no protection. Note the risk if it seems worth a follow-up. |
| Fan-out or parallel work the code actually performs | Bound concurrency, propagate cancellation and timeouts, keep shared state race-safe. |
| Multi-step change to an external system | See "External Workflows and Reconciliation". |

## Outbox

Rollback does not undo an external effect, so effects normally happen after commit. When losing the effect is unacceptable (the process may crash between commit and send), write an outbox row (a record of the intended effect) in the same transaction as the change, then deliver it from a separate dispatcher that marks it sent after the effect succeeds. Use the project's existing equivalent if it has one. Delivery is then at least once, so the receiver or the dispatcher must tolerate duplicates.

## Automatic Retries

When the system itself repeats work (retried jobs, redelivered messages, retry loops, an at-least-once dispatcher, a workflow resumed after a crash), make the second run harmless with:

- an idempotency key sent to the external service and stored with the record,
- a unique constraint that turns a duplicate into a detectable conflict,
- recoverable state (`pending`, `sent`, `failed`) that a later run can resume from.

Test the automatic retry and crash-between-steps paths, not only the happy path.

When a failure is returned to the user as a handled error and the user decides whether to try again, that retry policy is the project's choice. Do not add idempotency keys, request deduplication, or retry logic that was not asked for.

## Races

Protect a decision together with the change it guards when concurrent requests realistically contend for the same data and a wrong outcome causes real harm: overselling stock, overdrawing a balance, a duplicate that breaks a uniqueness invariant, or granting access that should be denied. Skip this for data one actor writes at a time, low-stakes last-write-wins fields, and tools that do not run concurrently.

Putting a read and a write in one transaction, especially at READ COMMITTED, does not stop another writer from invalidating the check. Prefer a conditional write (`UPDATE ... WHERE stock >= :qty`) or a unique constraint, and check the affected-row count. Otherwise use row locks or an isolation level that prevents the anomaly, and handle conflicts and serialization failures. A check done earlier in middleware or an earlier query is a hint, not a guarantee.

## Concurrency and Cancellation

When the code fans out over many items, use bounded concurrency (a worker pool, semaphore, or batch size) instead of one task per item. Pass cancellation and timeouts through to external calls so a cancelled request stops work. When in-memory state is actually shared across concurrent tasks or threads, protect it with the language's race-safe tools (locks, atomics, single-owner channels). Sequential code needs none of this.

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

## External Workflows and Reconciliation

Apply this section when code changes state in an external system (a gateway, proxy, DNS, cloud resource, identity provider, or provider API) in several steps, or through writes whose failure mode is unclear. A single idempotent call with clear success and failure semantics needs only the retry rules above.

| If this is true | Then |
| --- | --- |
| The change takes several external steps, or the process can crash between them | Persist the intent (desired state, operation id, steps completed) before the first external step, so a later run can resume or reconcile. Verify the final state with a matching read-back and a functional probe through the real path before finalizing. |
| A failed or ambiguous step can leave unsafe partial state (live credentials, open access, a half-applied route) | Reconcile toward the safe state for that workflow. A failed create or grant cleans up toward absence, removing only what this operation recorded creating. A failed delete, disable, or revoke keeps retrying toward revocation. |
| Leftovers are harmless, may hold data, or may have existed before this operation | Do not delete them. Retry idempotently from the recorded intent, or flag the workflow for review. |
| A write replaces the complete state (whole-document PUT, apply-all, replace-list), or a non-success response (error, timeout, 5xx) may still have changed live state | Treat the outcome as unknown. Do not finalize until a fresh successful write, a matching read-back, and a functional probe confirm it. A `/healthz` 200 or a management API 200 is not proof of readiness. |
| Several writers touch the same external state | Serialize them: one writer, a lock, a queue, or a compare-and-swap on a version or ETag. |
| Opening traffic, routes, listeners, or access depends on that state being correct | Keep the exposure closed until the live state is verified; open it last. |
| Always | Never reuse a privileged management credential as a client credential or as a fallback. Use a separate, least-privilege credential per role. |

When none of these conditions apply, a checked success response from the API's documented contract is enough. Do not add destructive cleanup or extra probes to workflows that do not need them.

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "The job retries automatically, so it is resilient." | A retry without an idempotency key, constraint, or recoverable state duplicates the effect. |
| "Add an idempotency key to every endpoint just in case." | Only where the system retries automatically or the project asks for it. |
| "I checked the balance earlier, so the write is safe." | On contended data, protect the check and write together; sharing a transaction is not enough. |
| "Two requests could theoretically interleave, so add a lock." | Only if concurrent writes are realistic and the wrong outcome is costly. |
| "The management API returned 200, so the new config is live." | Read back and probe the real path when writes replace complete state or failures may still mutate it. |
| "The request errored, so nothing changed." | A non-success write may have partially applied. Reconcile or verify. |
| "Clean up by deleting whatever looks leftover." | Delete only what this operation recorded creating, and only where absence is the safe state. |
| "Use the admin credential as a fallback so the client keeps working." | Never reuse a privileged credential as a client or fallback credential. |
| "Spawn a task per item; the runtime will cope." | Bound concurrency and propagate cancellation. |

## Red Flags

- A retry loop around a non-idempotent call.
- `SELECT` then `UPDATE` on contended, high-stakes data with no conditional write, lock, constraint, or adequate isolation.
- Finalizing an external change based only on a 200 or a health check when the write replaces complete state or failures may still mutate it.
- A privileged credential used as a client or fallback credential.
- Locks, idempotency keys, or reconciliation added to code with no concurrent writers and no automatic retries.
