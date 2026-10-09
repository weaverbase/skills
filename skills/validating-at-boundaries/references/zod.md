# Zod 4

How to apply the rules in [../SKILL.md](../SKILL.md) in TypeScript with Zod. Schemas own shape and validation; services and components receive validated, typed values.

Examples use Zod 4 (`z.uuid()`, `z.iso.datetime()`, `z.strictObject`).

## Schema and Derived Type

Define each concept's shape once. Name the schema `XSchema` and derive the type as `type X = z.infer<typeof XSchema>`. Do not write a second hand-made type.

```typescript
import { z } from "zod";

export const BookmarkSchema = z.object({
  id: z.uuid(),
  modelId: z.string().min(1),
  createdAt: z.iso.datetime().transform((value) => new Date(value)),
});

export type Bookmark = z.infer<typeof BookmarkSchema>;

export const CreateBookmarkSchema = z.strictObject(
  BookmarkSchema.pick({ modelId: true }).shape,
);

export type CreateBookmark = z.infer<typeof CreateBookmarkSchema>;
```

- Derive create, update, and public variants from the shared schema with `.extend()`, `.partial()`, `.pick()`, and `.omit()`. Do not copy field definitions.
- Use `z.strictObject` for inbound request bodies so unknown keys are rejected, when the project has no other convention. Plain `z.object` strips unknown keys, which is acceptable for external API responses where new upstream fields should not break you.
- Do not use `z.coerce.date()` for external data. It calls `new Date(value)`, so `null` silently becomes `1970-01-01` and other junk can become valid dates. Validate the string format and transform it, as `createdAt` does above.

## Update and Outbound Variants

Derive the update payload from the create schema, and validate outbound data with an explicit public schema so internal fields cannot leak.

```typescript
export const UpdateBookmarkSchema = CreateBookmarkSchema.partial();

const AccountSchema = z.object({
  id: z.uuid(),
  email: z.email(),
  passwordHash: z.string(),
});

export const PublicAccountSchema = AccountSchema.omit({ passwordHash: true });

// The default object behavior strips unknown keys, so passwordHash is dropped.
const body = PublicAccountSchema.parse(account);
```

## Parse at the Boundary

An adapter parses external data, and the component receives a typed prop.

```typescript
export async function fetchBookmarks(): Promise<Bookmark[]> {
  const response = await fetch("/api/bookmarks");
  return z.array(BookmarkSchema).parse(await response.json());
}

function BookmarkRow({ bookmark }: { bookmark: Bookmark }) {
  return <span>{bookmark.modelId}</span>;
}
```

Zod parsing belongs in route loaders, server actions, API-client adapters, form resolvers, or explicitly named boundary/container modules. Reusable and presentational components receive typed props. They do not accept `unknown` or `any`, call `parse`/`safeParse`, or silently hide invalid external data by returning `null`.

## Surfacing Validation Errors

The boundary turns a failed parse into a 400 response or a form error. Services raise domain errors, not validation errors.

```typescript
export async function createBookmarkHandler(request: Request): Promise<Response> {
  let payload: unknown;
  try {
    payload = await request.json();
  } catch (error) {
    if (error instanceof SyntaxError) {
      return Response.json({ error: "Invalid JSON" }, { status: 400 });
    }
    throw error;
  }
  const result = CreateBookmarkSchema.safeParse(payload);
  if (!result.success) {
    return Response.json(
      { errors: z.flattenError(result.error).fieldErrors },
      { status: 400 },
    );
  }

  // `result.data` is a validated CreateBookmark.
  return Response.json(await createBookmark(result.data), { status: 201 });
}
```

Branching on legitimately optional or empty values is domain logic and stays in the service. Do not repeat constraints the schema already guarantees.

## Environment Configuration

Parse `process.env` with a schema once at startup or boundary initialization. Export the typed result and use it everywhere else. Do not hardcode configuration values, and do not read `process.env` ad hoc inside services.

```typescript
const EnvSchema = z.object({
  DATABASE_URL: z.url(),
  API_KEY: z.string().min(1),
  REQUEST_TIMEOUT_S: z.coerce.number().positive().default(10),
});

export const env = EnvSchema.parse(process.env);
```

Present environment values are strings; missing values are `undefined`. Here `z.coerce.number().positive()` accepts numeric strings, rejects empty or invalid values, and applies the default only when the value is missing. For dates, still validate the string format before transforming it rather than relying on permissive date coercion.

## Bad: The Component Checks Shape Itself

```typescript
function BookmarkRow({ value }: { value: any }) {
  if (!value?.id || typeof value.id !== "string") {
    return null;
  }

  return <span>{value.modelId}</span>;
}
```

This accepts untyped input, repeats a check the schema should own, and hides invalid data by rendering nothing. Parse in a loader, adapter, or container module and pass a `Bookmark` instead.
