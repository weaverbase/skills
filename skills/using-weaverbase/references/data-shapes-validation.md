# Data Shapes Own Their Validation

Data models are the single source of truth for both shape and validation.
Services and components consume already validated, typed values. They should not
re-validate or sanitize data with ad hoc checks.

Schema validators own structural validation: types, required fields, length,
format, and coercion. Services may still enforce domain invariants that require
business context, persistence, authorization, or current system state.

In React, Zod parsing belongs in route loaders, server actions, API-client
adapters, form resolvers, or explicitly named boundary/container modules.
Reusable and presentational components receive typed props; they do not accept
`unknown` or `any`, call `parse`/`safeParse`, or silently hide invalid external
data by returning `null`.

## Rules

1. Define each shape once.
   - Python: use Pydantic `BaseModel` with `Field`, validators, and
     `ConfigDict`.
   - TypeScript: use a Zod schema and derive the type with `z.infer`.
2. Validate at boundaries: request bodies, query params, environment variables,
   external API responses, form inputs, and message payloads.
3. Services accept domain types, not raw `dict`, JSON, `unknown`, or `any`.
4. Do not duplicate structural constraints with checks like `if not value` or
   `len(value) > 50` inside services or components. Put those constraints on
   the model or schema.
5. Keep one schema per concept. Derive variants with explicit create/update
   models, Pydantic helpers such as `model_copy`, project helpers when they
   exist, or Zod methods such as `.partial()`, `.pick()`, and `.omit()`.
6. Surface validation errors at the boundary. Services raise domain errors, not
   validation errors.

## Pydantic

Good: the model owns validation, and the service accepts the validated type.

```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class CreateUser(BaseModel):
    model_config = ConfigDict(frozen=True)

    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


def create_user(data: CreateUser) -> User:
    ...
```

Bad: the service re-validates raw input.

```python
def create_user(data: dict) -> User:
    if "@" not in data.get("email", ""):
        raise ValueError("bad email")
    if len(data.get("name", "")) > 100:
        raise ValueError("name too long")
    ...
```

## Zod

Good: the schema owns validation, and the type is derived from it.

```typescript
import { z } from "zod";

export const Bookmark = z.object({
  id: z.string().uuid(),
  modelId: z.string().min(1),
  createdAt: z.coerce.date(),
});

export type Bookmark = z.infer<typeof Bookmark>;

const bookmark = Bookmark.parse(await response.json());
```

Bad: the component accepts untyped input and checks shape itself.

```typescript
function BookmarkRow({ value }: { value: any }) {
  if (!value?.id || typeof value.id !== "string") {
    return null;
  }

  return <span>{value.modelId}</span>;
}
```
