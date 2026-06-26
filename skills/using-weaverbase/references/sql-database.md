# SQL / Database

Use database types and access patterns that preserve correctness and async performance.

## PostgreSQL

Use `JSONB` for JSON data in PostgreSQL. Avoid `JSON` unless a specific compatibility constraint requires it.

Good:

```sql
metadata JSONB NOT NULL DEFAULT '{}'::jsonb
```

Bad:

```sql
metadata JSON NOT NULL DEFAULT '{}'
```

## Timestamps

Use `TIMESTAMPTZ` for timestamps. Avoid `TIMESTAMP` without timezone for application data.

Good:

```sql
created_at TIMESTAMPTZ NOT NULL DEFAULT now()
```

Bad:

```sql
created_at TIMESTAMP NOT NULL DEFAULT now()
```

## ORM and database queries

Use async ORM methods and async database clients in async applications, such as SQLAlchemy 2.0 async APIs or Prisma async calls. Avoid synchronous queries that block async request handlers, workers, or event loops.
