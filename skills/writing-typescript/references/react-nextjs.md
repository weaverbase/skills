# React and Next.js

Use function components and the App Router by default for new work, with hooks where needed. This reference covers framework conventions; for the base language rules (ES modules, `const`/`let`, `async`/`await`, array methods) see [../SKILL.md](../SKILL.md).

Checked against official React 19-era and Next.js 16.4 documentation on 2026-10-08. Check the project's installed versions and configuration before applying version-specific APIs; generator defaults are not the configuration of every existing app.

These rules apply to new code and requested changes. Do not migrate working Pages Router code, class components, state libraries, or caching configurations outside the request.

## Components

Use function components. In App Router, pages and layouts are Server Components by default. Add small Client Component boundaries for state, effects, event handlers, and browser APIs; components that only render data do not need state hooks.

Good (a standalone interactive App Router component):

```tsx
// app/ui/counter.tsx
"use client";

import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>;
}
```

`"use client"` declares a module-graph boundary, not browser-only execution: Client Components can still be prerendered on the server. Imported modules in that client graph do not each need the directive. Keep server-only code out of that graph, mark sensitive modules with `import "server-only"`, and pass React-serializable props across the server/client boundary. Server Components passed as children can remain server-rendered.

Bad (an ordinary counter implemented as a class):

```tsx
"use client";

import { Component } from "react";

export class Counter extends Component<{}, { count: number }> {
  state = { count: 0 };

  render() {
    return (
      <button onClick={() => this.setState({ count: this.state.count + 1 })}>
        Clicked {this.state.count} times
      </button>
    );
  }
}
```

A custom error boundary implemented with React's class lifecycle APIs is a legitimate exception. Next.js route error files (`error.tsx`) are function Client Components; using framework or library boundaries does not require writing your own class.

## Routing

Use the App Router (`app/`) for new Next.js projects. Maintain the existing router when changing a Pages Router area; its APIs are not valid in App Router work.

Good:

```text
app/orders/page.tsx
app/orders/[id]/page.tsx
```

Bad (for new work in an App Router project):

```text
pages/orders/index.tsx
pages/orders/[id].tsx
```

## Data Fetching

Fetch server data from Server Components through an API-client adapter or an existing query/operation function. Do not add an internal HTTP round trip just to call your own server-side data access. App Router does not use `getServerSideProps`, `getStaticProps`, or `getInitialProps`.

Choose caching deliberately for freshness and privacy. Keep private/user-specific data uncached by default; never share authenticated responses across users by copying a public-data example. Caching does not replace authentication or current authorization checks.

### With Cache Components

Cache Components was introduced in Next.js 16 and requires the Node.js runtime, not Edge routes. Recommended new apps generated with Next.js 16.4 enable it and Partial Prefetching. For an existing or manually configured 16.4 app adopting this model, the recommended configuration is:

```typescript
// next.config.ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  cacheComponents: true,
  partialPrefetching: true,
};

export default nextConfig;
```

Use `"use cache"` and `cacheLife` for deliberately cacheable functions or components. Do not read request APIs such as `cookies()` or `headers()` inside a shared `"use cache"` scope. For uncached async reads and runtime-dependent data, place the consuming subtree under `Suspense` so it can stream while the static shell renders; do not cache it just to remove the loading boundary. `Suspense` alone does not make synchronous content dynamic.

Good (public catalog data, using Cache Components and Zod 4):

```typescript
// app/lib/catalog.ts
import "server-only";
import { cacheLife } from "next/cache";
import { z } from "zod";

const ProductSchema = z.object({ id: z.uuid() });
const CatalogSchema = z.array(ProductSchema);
export type Product = z.infer<typeof ProductSchema>;

export async function getCatalog(): Promise<Product[]> {
  "use cache";
  cacheLife("hours");

  const response = await fetch("https://api.example.com/catalog");
  if (!response.ok) throw new Error("Catalog unavailable");
  return CatalogSchema.parse(await response.json());
}
```

```tsx
// app/catalog/page.tsx (Server Component)
import { getCatalog, type Product } from "../lib/catalog";

export default async function CatalogPage() {
  const products = await getCatalog();
  return <ProductList products={products} />;
}

function ProductList({ products }: { products: Product[] }) {
  return <ul>{products.map(product => <li key={product.id}>{product.id}</li>)}</ul>;
}
```

Parsing and HTTP error handling stay in the fetching adapter. Invalid responses propagate deliberately to error handling; a valid empty catalog renders normally. UUIDs are the sample API's contract, not a rule for all identifiers. Adapt the schema to the real source.

### Without Cache Components

Follow the installed version's fetch/route caching model. In Next.js 15+ without Cache Components, fetch responses are not stored in the persistent Data Cache by default, but a route can still be prerendered. Do not assume an unspecified `fetch` option guarantees fresh request-time rendering.

Explicit fetch options remain appropriate for this model:

```typescript
// Alternative options at the API-client boundary, not Cache Components guidance.
const publicResponse = await fetch("https://api.example.com/catalog", {
  next: { revalidate: 60 },
});
const freshResponse = await fetch("https://api.example.com/availability", {
  cache: "no-store",
});
```

Check `response.ok` and parse each external response before returning typed data, as in the adapter above. Choose route rendering deliberately too; do not mix previous-model options with Cache Components examples or mandate a caching migration for existing working code.

## Request APIs

Next.js 15 introduced asynchronous `params`, `searchParams`, `cookies()`, and `headers()`; Next.js 16 removed temporary synchronous compatibility. Await page props and request API results in server code. A synchronous Client Component can unwrap a provided promise with React's `use` API instead.

Good (runtime route params resolved beneath `Suspense`, also suitable with Cache Components):

```tsx
// app/catalog/[slug]/page.tsx
import { Suspense } from "react";

type RouteProps = { params: Promise<{ slug: string }> };

export default function ProductPage({ params }: RouteProps) {
  return (
    <Suspense fallback={<p>Loading product...</p>}>
      <ProductHeading params={params} />
    </Suspense>
  );
}

async function ProductHeading({ params }: RouteProps) {
  const { slug } = await params;
  return <h1>{slug}</h1>;
}
```

Use `await searchParams`, `await cookies()`, and `await headers()` at their consuming server boundary too. Page `searchParams` resolves to a plain object with `string | string[] | undefined` values, not a `URLSearchParams`. Parse route/query values against the actual input contract before passing them into operations; do not force arbitrary route IDs or slugs to be UUIDs.

## State Management

Distinguish local UI state, shared client state, and server data. Use `useState` or `useReducer` for local state; lift state or use context when appropriate. Follow established project conventions for shared state, whether context, Zustand, Jotai, or Redux Toolkit. Redux is not inherently obsolete; Redux Toolkit is the recommended approach for new Redux logic.

Keep server data in the server fetching layer or the project's client data-fetching/cache layer. Do not duplicate it into a global client store without a concrete need or add a new library just for a local filter.

Good:

```tsx
const [filter, setFilter] = useState<"all" | "open">("all");
```

Bad: introducing a global store and its plumbing solely for that filter. Using an already established store for genuinely shared state is not the same mistake.

## Actions and Forms

Use these React 19 APIs when their use case applies, not as replacements for all local state:

| API | Use for | Constraint |
| --- | --- | --- |
| `useActionState` from `react` | An Action's result/error state and pending status | The action receives previous state before its payload; dispatch through a form action or within `startTransition` |
| `useFormStatus` from `react-dom` | Pending submission UI inside a form | Call it in a child of the form, not the component that creates that same form |
| `useOptimistic` from `react` | Temporary optimistic UI while an Action is pending | Update within an Action/Transition, reconcile confirmed data, and surface failures |

React Actions can be client-side functions; Next.js Server Actions execute on the server. Server Actions must validate input at the boundary, authenticate the caller, and call operations that enforce current authorization and business rules. Return safe validation/domain errors for display; do not expose raw unexpected exceptions. Disabled buttons, optimistic state, and client validation are not security boundaries.

## Effects and Memoization

Use effects to synchronize with external systems, with cleanup where needed. Derive values from props/state during rendering rather than copying them into state via an effect. Keep user-triggered mutations in event handlers or Actions.

In React 19.2+, `useEffectEvent` separates genuinely non-reactive logic triggered by an effect while reading the latest committed props/state. Keep reactive dependencies in the effect; do not use it to silence dependency lint. Effect Events belong to their local effect logic: do not pass them to children, call them as ordinary click handlers, or include them in dependency arrays. Use compatible React Hooks lint rules.

When React Compiler is enabled, rely on its automatic memoization for new code instead of routinely adding `useMemo`, `useCallback`, or `memo`. Manual memoization remains available for precise control or measured needs. Preserve existing memoization unless removal is justified and tested; do not assume the compiler is enabled or require installing it for every project.

## Related Rules (Short Form)

- Parse API, form, and message payloads in a real boundary: route loaders, server actions, API-client adapters, form resolvers, or explicit container modules. Pass typed props to presentational components.
- Do not put `safeParse`, `if (!value?.id)`, or render-null shape guards inside reusable or presentational components; surface invalid data deliberately at the boundary.
- Keep route handlers, server actions, and UI event handlers thin; callable operations own data access, authorization, and business decisions.

## Red Flags

- "Add hooks to every component." Keep server-rendered UI on the server; add client boundaries only where needed.
- "Add `pages/` or `getServerSideProps` in this App Router feature." Follow the current router's APIs.
- "Copy this cache option into every fetch." Check version, configuration, freshness, and privacy first.
- "Cache these user orders like the public catalog." Keep private data uncached by default and enforce authorization independently.
- "Suppress effect dependencies with `useEffectEvent`." Keep dependencies that should trigger synchronization.
- "The submit button is disabled, so the action is protected." Enforce validation and authorization on the server.
- "Redux is old; replace the store." Follow the project's tools and actual state requirements without unrelated migrations.

## Common Mistakes

- Starting new features under `pages/` in a project that uses the App Router.
- Using `getServerSideProps`, `getStaticProps`, or `getInitialProps` in App Router work.
- Using state/effects without a client boundary, or treating Client Components as never server-rendered.
- Reading modern request APIs synchronously or resolving runtime data outside the required streaming boundary with Cache Components.
- Mixing caching models, assuming default fetch options guarantee freshness, or sharing private data through a public cache.
- Writing class components for ordinary UI or adding a global store for local state.
- Calling `useFormStatus` in the component that creates its form, or dispatching async/optimistic actions outside an Action/Transition.
- Using effects for derived values, hiding reactive dependencies, or adding/removing memoization without considering compiler configuration and behavior.
- Adding shape guards or `safeParse` inside a presentational component instead of validating at the boundary.
- Rewriting unrelated router, component, state-library, or caching code while making a small change.

## Official Sources

- [Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Caching with Cache Components](https://nextjs.org/docs/app/getting-started/caching), [configuration/runtime constraints](https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents), and [Next.js 16.4 defaults](https://nextjs.org/blog/next-16-4)
- [Caching without Cache Components](https://nextjs.org/docs/app/guides/caching-without-cache-components)
- [Page props](https://nextjs.org/docs/app/api-reference/file-conventions/page) and [Next.js 16 async request APIs](https://nextjs.org/docs/app/guides/upgrading/version-16)
- [useActionState](https://react.dev/reference/react/useActionState), [useFormStatus](https://react.dev/reference/react-dom/hooks/useFormStatus), and [useOptimistic](https://react.dev/reference/react/useOptimistic)
- [useEffectEvent](https://react.dev/reference/react/useEffectEvent) and [React Compiler](https://react.dev/learn/react-compiler/introduction)
- [Modern Redux with Redux Toolkit](https://redux.js.org/introduction/why-rtk-is-redux-today)
