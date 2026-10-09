---
name: designing-sql-schemas
description: Use when writing or changing SQL schemas, table definitions, migrations, or queries; choosing PostgreSQL column types for JSON or timestamps; choosing a database driver or ORM API; building SQL that includes user input; or planning a migration that drops, truncates, narrows, or rewrites existing data.
---

# Designing SQL Schemas

## Overview

Choose database types, access APIs, and migration steps that preserve correctness and schema history.

These rules apply to new or requested changes. Do not rewrite existing working schemas or queries solely to match them.

## Quick Reference

| Topic | Do | Do not |
| --- | --- | --- |
| JSON columns (PostgreSQL) | `JSONB` | `JSON`, unless a specific compatibility constraint requires it |
| Timestamps that identify instants (PostgreSQL) | `TIMESTAMPTZ` | Timezone-less `TIMESTAMP` for an instant |
| Database access API | Async API by default; in Go, context-aware calls | Blocking calls in async code |
| Applied migrations | Add a new migration | Edit a migration that has already been applied |
| Changes that can lose data | Additive steps, and state what is lost | A silent drop, truncate, or narrowing |
| User-supplied values in SQL | Bound parameters or ORM bindings | String-built SQL |

## Types

Use `JSONB` for JSON data in PostgreSQL.

```sql
metadata JSONB NOT NULL DEFAULT '{}'::jsonb
```

Use `TIMESTAMPTZ` for timestamps that identify instants, such as creation and expiration times. A local wall-clock value or schedule may deliberately use `TIMESTAMP` without time zone; define its timezone interpretation separately.

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

## Database Access API

When the user or project has not chosen otherwise, prefer the async database API: SQLAlchemy 2.0 async sessions with `asyncpg` or `psycopg` async in Python, `sqlx` or `tokio-postgres` in Rust, and promise-based clients such as Prisma, Drizzle, or `pg` in TypeScript. Never call a synchronous driver from async code; it blocks the event loop.

Go has no async/await. The equivalent is context-aware calls (`QueryContext`, `ExecContext`, `BeginTx`, or `pgx` methods that take `ctx`), passing the request's context so cancellation and timeouts reach the database.

Follow an existing project's choice. If it already uses a synchronous stack, keep it consistent rather than mixing both.

```python
async def get_order(session: AsyncSession, order_id: UUID) -> Order | None:
    result = await session.execute(select(Order).where(Order.id == order_id))
    return result.scalar_one_or_none()
```

```go
func GetOrder(ctx context.Context, db *pgxpool.Pool, id uuid.UUID) (Order, error) {
	var o Order
	err := db.QueryRow(ctx, "SELECT id, status FROM orders WHERE id = $1", id).Scan(&o.ID, &o.Status)
	return o, err
}
```

## Migrations

- Change the schema with a new migration. Never edit one that has already been applied: other environments already ran the old version and will silently diverge. Fix mistakes forward.
- For changes that can lose data (dropping a table or column, truncating, narrowing a type, adding a constraint existing rows may violate), prefer additive steps: add the new shape, backfill, switch readers, then drop. State in the migration or the change summary what data the destructive step removes.

## Parameterized SQL

Pass user-supplied and other untrusted values as bound parameters. Table and column names cannot be bound; pick them from a fixed allowlist in code.

Good:

```python
await session.execute(
    text("SELECT id, status FROM orders WHERE customer_id = :customer_id"),
    {"customer_id": customer_id},
)
```

Bad:

```python
await session.execute(text(f"SELECT id, status FROM orders WHERE customer_id = '{customer_id}'"))
```

## Common Mistakes

| Temptation | Decision |
| --- | --- |
| "`JSON` or `TIMESTAMP` is close enough." | Use `JSONB` and `TIMESTAMPTZ` unless a concrete constraint requires otherwise. |
| "A sync driver is simpler, and it is only one call." | Default to the async API (context-aware in Go). One blocking call in async code still blocks the event loop. |
| "Just edit the old migration; it is a small fix." | Add a new migration. |
| "Drop the old column in the same step." | Add, backfill, switch, then drop, and say what is removed. |
| "The value comes from our own UI, so concatenating is fine." | Bind it. Allowlist identifiers such as sort columns. |
