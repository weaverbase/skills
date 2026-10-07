# React and Next.js

Use function components with hooks and the App Router by default for new work. This reference covers syntax and framework conventions only; for the base language rules (ES modules, `const`/`let`, `async`/`await`, array methods) see [../SKILL.md](../SKILL.md).

These rules apply to new code and requested changes. Do not migrate working Pages Router code or class components that are outside the request.

## Components

Use function components with hooks. Avoid class components unless a framework constraint specifically requires one, such as an error boundary.

Good:

```tsx
import { useState } from "react";

export function Counter() {
  const [count, setCount] = useState(0);

  return <button onClick={() => setCount(count + 1)}>Clicked {count} times</button>;
}
```

Bad:

```tsx
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

An error boundary is the example of a genuine framework constraint: it still requires a class component, so a class is acceptable there.

## Routing

Use the App Router (`app/`) for new Next.js 13+ work. Avoid adding new Pages Router (`pages/`) code unless you are maintaining an existing Pages Router area.

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

Use Server Components and `fetch` with caching in App Router projects. Avoid `getServerSideProps`, `getStaticProps`, and `getInitialProps` for new App Router work.

Good:

```tsx
// app/orders/page.tsx (Server Component)
import { z } from "zod";

const ordersSchema = z.array(z.object({ id: z.string().uuid() }));
type Order = z.infer<typeof ordersSchema>[number];

export default async function OrdersPage() {
  const response = await fetch("https://api.example.com/orders", {
    next: { revalidate: 60 },
  });
  if (!response.ok) throw new Error("Orders unavailable");
  const orders = ordersSchema.parse(await response.json());

  return <OrderList orders={orders} />;
}

function OrderList({ orders }: { orders: Order[] }) {
  return <ul>{orders.map(order => <li key={order.id}>{order.id}</li>)}</ul>;
}
```

Bad:

```tsx
// pages/orders.tsx
export async function getServerSideProps() {
  const response = await fetch("https://api.example.com/orders");
  return { props: { orders: await response.json() } };
}
```

Parse external responses at the fetching boundary before passing typed props to components. Choose caching deliberately for the data; do not cache private or rapidly changing responses merely to copy this example.

## State Management

Use React hooks such as `useState`, `useReducer`, and `useContext`, or modern lightweight libraries such as Zustand or Jotai. Avoid Redux for simple state management unless project complexity justifies it.

Good:

```tsx
const [filter, setFilter] = useState<"all" | "open">("all");
```

Bad (a Redux store, actions, and reducers added only to hold one filter value):

```tsx
const filter = useSelector((state: RootState) => state.orders.filter);
const dispatch = useDispatch();
dispatch(setOrdersFilter("open"));
```

## Related Rules (Short Form)

- Parse API, form, and message payloads in a real boundary: route loaders, server actions, API-client adapters, form resolvers, or explicit container modules. Pass typed props to presentational components.
- Do not put `safeParse`, `if (!value?.id)`, or render-null shape guards inside reusable or presentational components; surface invalid data deliberately at the boundary.

## Red Flags

- "Use a class component because it is familiar." Only a framework constraint such as an error boundary justifies one.
- "Add a `pages/` route; it is quicker." New App Router work uses `app/`.
- "Use `getServerSideProps` here." Fetch in a Server Component instead.
- "Add Redux for this one value." Use hooks or a lightweight library unless complexity justifies Redux.
- "The legacy style still works." Working code is not a reason to start new code in the old style; a legacy reason is a concrete tool or platform constraint.

## Common Mistakes

- Starting new Next.js 13+ features under `pages/` in a project that uses the App Router.
- Using `getServerSideProps`, `getStaticProps`, or `getInitialProps` in App Router work.
- Writing a class component for ordinary UI instead of a function component with hooks.
- Introducing Redux for state that `useState`, `useReducer`, or `useContext` already covers.
- Adding shape guards or `safeParse` inside a presentational component instead of validating at the boundary.
- Rewriting existing, unrelated Pages Router or class-component code while making a small change.
