# Node.js

Use explicit Node core imports and promise-based APIs.

## Package imports

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

## File operations

Use promise-based APIs from `node:fs/promises`. Avoid callback-based APIs and synchronous file operations in application code.

Good:

```typescript
const contents = await fs.readFile(filePath, "utf8");
```

Bad:

```typescript
import fs from "node:fs";

const contents = fs.readFileSync(filePath, "utf8");
```
