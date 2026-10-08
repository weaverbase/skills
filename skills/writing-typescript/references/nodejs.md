# Node.js

Use explicit Node core imports and promise-based APIs. For the base language rules (ES modules, `async`/`await`) see [../SKILL.md](../SKILL.md).

Checked against official Node.js documentation on 2026-10-08: Node.js 22/24 are LTS, 26 is Current, and 20 is EOL. Check the project's `engines` and deployed runtime before using version-specific APIs; use a supported LTS release for production.

These rules apply to new code and requested changes. Do not rewrite unrelated working code to match them.

## Package Imports

Use the `node:` protocol for Node.js built-in modules. Avoid importing core modules without the protocol.

Good:

```typescript
import { readFile } from "node:fs/promises";
import { join } from "node:path";
```

Bad:

```typescript
import { readFile } from "fs/promises";
import { join } from "path";
```

The prefix makes it obvious that the import is a built-in and not a package from `node_modules`.

Named or namespace imports work without synthetic-default-import assumptions. Node's native ES modules also support default imports of these built-ins; TypeScript acceptance depends on the project's module/interoperability configuration. Preserve an existing working default-import style.

## File Operations

Use promise-based APIs from `node:fs/promises` for ordinary asynchronous file operations. Avoid synchronous file operations in request handlers, workers, or other concurrent paths: they block the executing thread's event loop. Synchronous calls are acceptable in one-shot CLIs, build scripts, and startup-only code before serving requests when blocking is deliberate.

Prefer promises over callbacks for new sequential async code. Callback APIs are still supported and asynchronous, not obsolete or blocking; use them when a concrete API requirement or measured performance need justifies it.

Good:

```typescript
import { readFile } from "node:fs/promises";

const contents = await readFile(filePath, "utf8");
```

Bad (in a request handler or concurrent worker):

```typescript
import { readFileSync } from "node:fs";

const contents = readFileSync(filePath, "utf8");
```

Avoid for ordinary sequential async work (callback style):

```typescript
import { readFile } from "node:fs";

readFile(filePath, "utf8", (error, contents) => {
  // ...
});
```

Import from `node:fs/promises` when only promise-based calls are needed. Use `node:fs` for APIs such as `createReadStream`/`createWriteStream`, or justified synchronous/callback calls; constants are also available from `node:fs/promises`.

For large files or streaming transformations, use streams with `pipeline` from `node:stream/promises` rather than loading the whole file with `readFile`. Await the pipeline so failures propagate and backpressure is respected.

Do not call `access()` to check existence or permissions before `open`/`readFile`/`writeFile`: the file can change between calls. Attempt the operation directly and handle the relevant error, such as `ENOENT` for a missing file; do not swallow unrelated errors.

## Current Idioms

- **Module paths:** In file-based ES modules, use `import.meta.dirname` and `import.meta.filename` instead of CommonJS `__dirname`/`__filename` or manual `fileURLToPath` boilerplate. They are stable in Node.js 22.16+/24+. For a module-relative resource, `new URL("./data.json", import.meta.url)` can be passed directly to file APIs.
- **Globbing:** Use `glob` from `node:fs/promises` when its supported patterns/options meet the need, rather than adding a dependency. It is stable in Node.js 22.17+/24+ and returns an async iterator consumed with `for await...of`, not an array-returning promise. Preserve an existing glob library unless replacement is requested.
- **Child processes:** Use `execFile` or `spawn` from `node:child_process` with a trusted executable, an argument array, and no shell (`shell: false`, the default). Do not interpolate untrusted values into `exec` command strings or enable a shell to defeat this protection. Validate arguments and paths too; argument arrays do not prevent the target program from interpreting input as options (use `--` when supported).

### Native TypeScript Execution

Apply this guidance only when the project runs TypeScript directly with Node's built-in type stripping, not when it bundles, compiles with `tsc`, or uses a loader such as `tsx`. Type stripping is enabled by default in Node.js 22.18+/24+ and stable in 24.12+/26+. It does not typecheck; keep a separate typecheck step.

Use erasable syntax and explicit type-only imports (`import type` or inline `type` specifiers). Type stripping cannot handle `enum`, runtime namespaces, constructor parameter properties, TypeScript import aliases (`import X = ...`), or decorators. Relative imports need the actual source extension, such as `./operation.ts`; `.tsx` and TypeScript under `node_modules` are unsupported. Node ignores `tsconfig.json`, including `paths` aliases; use package `#` subpath imports if aliases are needed.

For TypeScript 5.8+ checking this execution mode, use `module: "nodenext"`, `erasableSyntaxOnly`, and `verbatimModuleSyntax`; `rewriteRelativeImportExtensions` supports emitting JavaScript with rewritten relative extensions when needed. Do not impose these settings, `.ts` imports, or type stripping on projects using another execution/build model.

## Related Rules (Short Form)

- Read environment variables through a schema validated once at startup, not as scattered raw `process.env` reads.
- Do not build shell commands from untrusted strings, and validate file paths that come from user input before reading or writing them.

## Red Flags

- "Import `fs` without `node:` because it is shorter."
- "`readFileSync` is simpler for this request handler."
- "A callback-style `fs` call is fine because the old example used it."
- "Build an `exec` command by concatenating the submitted filename."
- "Use `readFile` for this unbounded upload; memory will be fine."
- "Plain Node will run any TypeScript feature and resolve our `tsconfig` aliases."
- "The legacy style still works." A legacy reason is a concrete tool or platform constraint, not familiarity.

## Common Mistakes

- Importing `fs`, `path`, or other built-ins without the `node:` prefix.
- Using `*Sync` file APIs in request handlers, workers, or other concurrent paths, or defaulting to callbacks for ordinary sequential async file work without a concrete reason.
- Importing from `node:fs` when only promise-based calls are needed, instead of `node:fs/promises`.
- Interpolating untrusted input into shell commands, or treating argument arrays as input validation.
- Reading a large file fully into memory when streaming fits the operation.
- Using non-erasable syntax or extensionless imports in code executed through native type stripping, or treating execution as a typecheck.
- Rewriting unrelated working imports or glob libraries while making a small change.

## Official Sources

- [Node.js releases and support status](https://nodejs.org/en/about/previous-releases)
- [File system APIs, globbing, and access-check races](https://nodejs.org/api/fs.html)
- [ES modules, built-in exports, and module paths](https://nodejs.org/api/esm.html)
- [Child processes and shell behavior](https://nodejs.org/api/child_process.html)
- [Streams and promise-based pipelines](https://nodejs.org/api/stream.html)
- [Native TypeScript execution and limitations](https://nodejs.org/api/typescript.html)
