# Node.js

Use explicit Node core imports and promise-based APIs. For the base language rules (ES modules, `async`/`await`) see [../SKILL.md](../SKILL.md).

These rules apply to new code and requested changes. Do not rewrite unrelated working code to match them.

## Package Imports

Use the `node:` protocol for Node.js built-in modules. Avoid importing core modules without the protocol.

Good:

```typescript
import fs from "node:fs/promises";
import path from "node:path";
```

Bad:

```typescript
import fs from "fs/promises";
import path from "path";
```

The prefix makes it obvious that the import is a built-in and not a package from `node_modules`.

## File Operations

Use promise-based APIs from `node:fs/promises`. Avoid callback-based APIs and synchronous file operations in application code.

Good:

```typescript
import fs from "node:fs/promises";

const contents = await fs.readFile(filePath, "utf8");
```

Bad:

```typescript
import fs from "node:fs";

const contents = fs.readFileSync(filePath, "utf8");
```

Also bad (callback style):

```typescript
import fs from "node:fs";

fs.readFile(filePath, "utf8", (error, contents) => {
  // ...
});
```

## Related Rules (Short Form)

- Read environment variables through a schema validated once at startup, not as scattered raw `process.env` reads.
- Do not build shell commands from untrusted strings, and validate file paths that come from user input before reading or writing them.

## Red Flags

- "Import `fs` without `node:` because it is shorter."
- "`readFileSync` is simpler for this request handler."
- "A callback-style `fs` call is fine because the old example used it."
- "The legacy style still works." A legacy reason is a concrete tool or platform constraint, not familiarity.

## Common Mistakes

- Importing `fs`, `path`, or other built-ins without the `node:` prefix.
- Using callback-based or `*Sync` file APIs in request handlers, workers, or other application code.
- Importing from `node:fs` when only promise-based calls are needed, instead of `node:fs/promises`.
- Rewriting unrelated working imports while making a small change.
