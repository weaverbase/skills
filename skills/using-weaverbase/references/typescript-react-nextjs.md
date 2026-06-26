# TypeScript, React, and Next.js

Use modern ESM, async syntax, functional React, and App Router patterns by default.

## TypeScript / JavaScript

### Module imports

Use ES module `import`/`export` syntax. Avoid CommonJS `require()` and `module.exports`.

Good:

```typescript
import { parseInvoice } from "./parseInvoice";

export { parseInvoice };
```

Bad:

```typescript
const { parseInvoice } = require("./parseInvoice");

module.exports = { parseInvoice };
```

### Variable declarations

Use `const` by default and `let` when reassignment is needed. Do not use `var`.

### Async operations

Use `async`/`await` for asynchronous control flow. Avoid raw `.then()` chains except when they are the clearer primitive for a specific API or composition pattern.

Good:

```typescript
const response = await fetch(url);
const bookmark = Bookmark.parse(await response.json());
```

Bad:

```typescript
fetch(url).then((response) => response.json()).then((value) => value);
```

### Array transformations

Use modern array methods such as `.map()`, `.filter()`, `.reduce()`, and `.find()` for transformations and lookups. Avoid traditional `for` loops when an array method communicates the intent better.

## React / Next.js

### Components

Use function components with hooks. Avoid class components unless a framework constraint specifically requires one, such as an error boundary.

### Routing

Use the App Router (`app/`) for new Next.js 13+ work. Avoid adding new Pages Router (`pages/`) code unless maintaining an existing Pages Router area.

### Data fetching

Use Server Components and `fetch` with caching in App Router projects. Avoid `getServerSideProps`, `getStaticProps`, and `getInitialProps` for new App Router work.

### State management

Use React hooks such as `useState`, `useReducer`, and `useContext`, or modern lightweight libraries such as Zustand or Jotai. Avoid Redux for simple state management unless project complexity justifies it.
